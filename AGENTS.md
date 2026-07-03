# Agent Guide

Auto-loaded by opencode as project-level instructions (see `opencode.json`
-> `instructions`). Extend this file as the project evolves — especially
once `rebus`'s Go source exists and there are real build/test/lint commands
to record.

## Project

`rebus` (`github.com/monamaret/rebus`) is the **private 1:1 chat app's
backend** — one application within the personal platform that `skipper`
scaffolds and operates. It is **not** the platform tool, a TUI shell, or a
client; it is a single deployable backend with a published client library.

- **Shape:** a stateless RPC-over-HTTPS backend in the rook/sight mold — no
  open per-client session, no server-side TUI rendering. Written in **Go**.
- **Storage:** Cloud Firestore,
  `conversations/{conversationId}/messages/{messageId}` subcollections;
  `updatedAt`-incremental sync (extending `sight`'s `SyncNotes(ctx, since)`
  pattern). Clients never touch Firestore directly — all access is
  backend-RPC-mediated.
- **Confidentiality:** transport (HTTPS) + Firestore at-rest encryption
  only — **not** end-to-end encrypted; backend can read `content`.
  Indefinite retention. Async-only delivery (no live/real-time mode).
  Multiple independent 1:1 conversations.
- **Identity/auth:** SSH-public-key-signed challenge; the private key never
  leaves the client; IdP-free (independent of GCP/Google identity).
  Outside-domain participants provisioned via the owner-controlled public-key
  registry. WebAuthn is reserved for a future web client only.
- **Roles:** asymmetric, enforced **server-side** — admin (owner) can
  hard-delete/edit/invite; participant can only hide (`hiddenFor`). Admin
  operations surface only inside `kingfish`'s embedded owner view — never in
  `bateau`, never as a `skipper` command.
- **Clients (v1, TUI-only):** both share `rebus`'s single public Go client
  package — `kingfish`'s embedded owner-side chat view (full admin) and
  [github.com/monamaret/bateau](https://github.com/monamaret/bateau) (the
  guest's lighter standalone client — send/read/hide only).
- **Deployment:** to GKE via `skipper app add --github`; own namespace
  `skipper-rebus` (default-deny `NetworkPolicy`) and own GCP IAM service
  account `skipper-rebus` (Workload Identity). This repo builds from source
  via WIF-authenticated CI into Artifact Registry; the persisted Kustomize
  base + Config Connector IAM/namespace config live in **`skipper`'s**
  `deployments/rebus/`, not here.

The design context above is settled in the vendored research corpus under
`specs/research/` (copied from `skipper`) — chiefly `006` (chat design),
`013` (privilege model), `014` (guest client/auth), `019` (namespace), `021`
(IAM), `023` (repo inventory), and `026` (client-package contract). See
[.specify/memory/constitution.md](.specify/memory/constitution.md) for the
non-negotiable principles governing this repo. This is a private, non-public
repo holding real conversation-service configuration over time; it is not
designed for reuse outside this context.

## Conventions

- **Language/stack:** Go, following standard project layout, wrapped errors
  (`fmt.Errorf("...: %w", err)`), structured logging (not raw
  `fmt.Println` for operational output), and `context.Context` propagation
  through all I/O and RPC calls. (Framework specifics — e.g. the RPC/HTTP
  and Firestore client libraries — are recorded here once decided via the
  research/decision workflow below.)
- **Backend surface:** each operation is a `func(ctx, Input) (Output, error)`
  (per `011`'s shared-operation convention); `context.Context` is always the
  first parameter. Delivery is async/pull only — do not introduce
  live/streaming/push paths (Constitution II).
- **Public client package (a cross-repo contract):** exactly one public
  package, `github.com/monamaret/rebus/client`, exporting the `Client`, its
  `New(cfg Config)` constructor, the request/response wire types, and the
  error sentinels — **never** under `internal/` (cross-module import must
  compile). It follows the shared shape in
  [specs/research/026-public-client-package-contract.md](specs/research/026-public-client-package-contract.md).
  Any breaking change to this surface is a **SemVer-major** release,
  coordinated with `kingfish` and `bateau`. `rebus`'s own method set is
  finalized in feature item **F-005**.
- **Data model:** Firestore `conversations/{id}/messages/{id}`; message docs
  carry `sender`/`content`/`sentAt`/`updatedAt`/`hiddenFor`; conversation
  docs carry the two participant identities (keyed by public key), a
  per-participant `role`, and `createdAt`. Sync is
  `messages.where("updatedAt", ">", lastSyncTime)`. No relational or
  self-hosted store.
- **Auth:** SSH-key-signed challenge only; never transmit the private key;
  no IdP dependency. Enforce roles at the backend/operation layer, not in
  clients.
- **GCP/GKE:** Workload Identity (workload) and Workload Identity Federation
  (CI/CD) — **no static service-account keys committed anywhere**. Secrets in
  Secret Manager. Images in Artifact Registry. Deployment manifests
  (Kustomize base, Config Connector IAM/namespace) live in `skipper`'s
  `deployments/rebus/`, not in this repo (repo-separation).
- **Test-driven development (NON-NEGOTIABLE):** write a failing test before
  the implementation that makes it pass — red-green-refactor, for every
  feature, fix, and refactor, across the backend operations, the Firestore
  access/sync logic, and the published `client` package. No change lands
  without a test demonstrating the gap it closes.
- **No speculative abstraction:** single-owner backend, two known
  owner-controlled consumers — favor the minimal-dependency path; do not
  generalize toward group chat, federation, real-time, multi-tenant, extra
  providers, or E2EE (any of which is a constitution-level change, not a
  feature).

## Build / Test / Lint

Not yet established — `rebus`'s Go module does not exist yet. Once it does,
record the real commands here (e.g. `go build ./...`, `go test ./...`, a
`./scripts/check.sh` for `gofmt`/`go vet`/`staticcheck`, and whatever the
Firestore-emulator-backed integration tests need). No change lands without
these commands passing, per the test-driven-development convention above.

## Subagents

None yet — `.opencode/agents/` only contains its own `README.md` (format
reference for defining project-scoped subagents). Add entries here, one line
per agent, as they're created.

## Skills

Beyond the Spec Kit-generated `.claude/skills/speckit-*/` command surface:

- **`tech-research`** — research a technical question/library choice/
  architecture decision following the standardized `specs/research/` pattern
  (library/best-fit assessment, existing-implementation survey, mandatory
  Web Citations section). Use before a topic becomes a feature item.
- **`release-tracking`** — gates implementation work on it tracing to an
  existing `specs/features/` item, roadmap entry, or issue; generates the
  release directory and a standardized `CHANGELOG.md` entry (from
  `specs/releases/CHANGELOG-TEMPLATE.md`) when a release closes.

Add entries here, one per skill, as more project-specific skills are created.

## Feature development process

The fixed pipeline from idea to shipped capability — steps are not skipped,
reordered, or collapsed, even for small features:

1. **Numbered research discussion document** — open or extend a numbered doc
   under `specs/research/` (copy `specs/research/research-template.md` for a
   new one) and record confirmed decisions in its Decision Log (date,
   decision, rationale, owner; superseded entries kept, marked
   **Superseded**, not deleted). The docs presently under `specs/research/`
   are the vendored platform corpus copied from `skipper`; `rebus`'s own new
   research extends it rather than replacing it.
2. **Feature item creation** — create a feature item under `specs/features/`
   from `specs/features/TEMPLATE.md`, pointing back to the confirmed
   decisions rather than re-deciding anything inline. (`rebus`'s
   operation/method set is finalized in **F-005**.)
3. **Roadmap inclusion** — add the feature item to
   [specs/research/roadmap.md](specs/research/roadmap.md) in its
   dependency-ordered place.
4. **Architecture sync** — update
   [specs/research/architecture.md](specs/research/architecture.md) with any
   resulting diagram/call-flow changes *before* implementation starts.
5. **Spec Kit workflow** — only then run the Spec Kit (speckit) workflow
   (`/speckit.specify` → `/speckit.clarify` → `/speckit.plan` →
   `/speckit.tasks` → `/speckit.implement`, etc.) to produce spec/plan/tasks
   artifacts and drive implementation, following the documented process at
   https://github.com/github/spec-kit/blob/main/spec-driven.md rather than
   improvising a different artifact shape.

**On the vendored research numbering.** `specs/research/NNN-*.md` were copied
from `skipper` and keep `skipper`'s numbering, which is platform-wide (e.g.
`006` is the chat design, `013` the privilege model). When referring to one
in prose, write **`R-NNN`** for a research doc and **`F-NNN`** for a feature
item, and always path-qualify the link — never use a bare `[NNN]`.

## Release process

Each release is packaged as a directory `specs/releases/X.XX-name/` once its
features have actually shipped — not drafted ahead of the work it documents.
It MUST contain:

- **Final feature list** — each entry linking back to its `specs/features/`
  item (and from there to the research doc/Decision Log entries that
  justified it).
- **Release notes** — a plain, owner-facing summary of what changed and why
  (no external-user/marketing framing — this is a private, single-owner
  project).
- **Unit and integration tests** — both layers, covering the release's
  capabilities end to end, per the test-driven-development convention above;
  a release isn't "final" until both exist and pass.
- **Usage guide** — how to build, configure, deploy (via `skipper`), upgrade,
  and maintain `rebus` at that release, plus which `client`-package SemVer
  importers should consume.

Because the `client` package is a versioned cross-repo contract, every
release states its `client`-package SemVer and calls out any breaking change
explicitly. Every release also adds an entry to the root `CHANGELOG.md` using
the scaffold in `specs/releases/CHANGELOG-TEMPLATE.md` (one line item per
shipped capability, linked back to its `specs/features/` item; a **Deferred**
category for anything pushed out of scope). The `release-tracking` skill
automates this step and the scope-alignment check that should gate
implementation work in the first place.

## Notes

- Treat [.specify/memory/constitution.md](.specify/memory/constitution.md) as
  the source of truth for non-negotiable principles, and
  [specs/research/roadmap.md](specs/research/roadmap.md) for platform task
  status.
- `rebus` is deployed and operated **by `skipper`**, not by itself:
  `skipper app add --github` builds this repo and applies the Kustomize/IAM
  config that lives in `skipper`'s `deployments/rebus/`. This repo owns the
  Go source, the container build, and the WIF-authenticated CI that pushes to
  Artifact Registry — not the deployment manifests.
- No static GCP service-account keys or other long-lived credentials are ever
  committed to this repo — see the constitution's Security & Operational
  Posture section.
- The confidentiality model is **transport + at-rest, not E2EE**, retention
  is **indefinite**, and roles are enforced **server-side** — these are
  deliberate, recorded decisions (`006`/`013`). Adopting E2EE, finite
  retention, client-side role checks, group chat, or a real-time mode is a
  constitution-level change, not a routine feature — don't drift into them.
- Related repos (all separate, most not yet created):
  `github.com/monamaret/bateau` (guest client, imports `rebus/client`),
  `kingfish` (TUI shell, embeds the owner chat view via `rebus/client`),
  `skipper` (the platform scaffold/deploy tool that operates `rebus`). Their
  source, specs, and references live in their own repos, not here.

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->
