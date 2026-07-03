# Research: Private chat application — detailed requirements and design

**Status:** Complete — discovery interview fully answered; storage/transport/client-sharing research done; ready to update specs/features/005-private-chat-1to1.md with concrete specifics. All four follow-up items flagged earlier are now resolved by sibling docs (009, 013, 014) — see Open Questions.
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

The platform-level architecture for the private chat app is already
decided — see Background & Context below. What hasn't been researched yet
is the chat application's own detailed design: its confidentiality model,
message lifecycle, delivery/transport mechanics, multi-conversation scope,
and notification UX. These are genuinely open and shape real
implementation choices (e.g. end-to-end encryption changes the entire
data model), so this doc starts as a discovery interview rather than a
library/pattern survey — the survey work happens *after* the interview
answers narrow the actual design space.

- **Goal:** Turn the owner's answers to the discovery interview (Appendix)
  into a concrete chat-application design: confidentiality model, message
  storage/lifecycle, transport, conversation scope, notification UX —
  narrow enough to update
  [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md)'s
  Scope with real specifics instead of placeholders.
- **In scope:** Everything the interview covers — security/confidentiality
  model, message lifecycle (retention, delete/edit, export), delivery and
  transport mechanics, identity/multi-conversation scope, notification and
  TUI UX, the outside-domain participant's onboarding experience, and any
  compliance/retention constraints.
- **Out of scope:** Re-deciding what's already confirmed — the
  stateless rook/sight interaction model, the simple-custom-protocol (not
  IRC) choice, SSH-public-key-based outside-domain provisioning, and
  TUI-only-for-v1 scope are all settled (see Background & Context) and are
  not reopened here.
- **Success criteria:** Every interview question answered and reflected in
  this doc's Decision Log, with [005-private-chat-1to1.md](../features/005-private-chat-1to1.md)
  updated to match.
- **Non-goals:** Building or even fully comparing candidate
  libraries/protocols before the interview answers are in — guessing at
  the design now would waste the comparative-research effort if the
  answers point a different direction.

## Background & Context

- **Relevant prior work (already confirmed, not reopened here):**
  - [002-hybrid-tui-layer.md → Decision Log](002-hybrid-tui-layer.md#decision-log) —
    chat uses the stateless rook/sight pattern (not bbb-le's SSH-session-
    served model) specifically for this 1:1, cross-surface app; simple
    custom message protocol, not embedded IRC; outside-domain provisioning
    is SSH-public-key-based via the owner-controlled registry, independent
    of any GCP/Google identity; WebAuthn/passkeys are the decided
    mechanism for *if/when* a web client is added.
  - [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md) —
    current feature scope: TUI-only for v1, web client deferred to the
    generic app scaffold ([R-005](005-generic-app-scaffold.md)/[F-006](../features/006-deployment-admin-dashboard.md))
    if/when chat ever adopts it.
  - [specs/research/002-hybrid-tui-layer.md → Security & Compliance](002-hybrid-tui-layer.md#security--compliance) —
    no long-lived credentials leave the client; admin role separate from
    chat access; explicit key-revocation tooling required.
- **Domain assumptions:** "Stateless" so far has meant *the backend
  doesn't hold an open session per client* (rook/sight pattern) — it does
  **not** yet specify whether message *content* is readable by the
  backend (server-side encryption only) or never readable by anything but
  the two participants (end-to-end encryption). That distinction is the
  single biggest open question and the interview leads with it.
- **Stakeholders:** mona (owner/operator, one of the two chat
  participants) and the invited outside-domain participant (the other).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **Cloud Firestore** | Conversation/message storage | Proprietary GCP managed service | Mature (GA since 2017) | Actively maintained by Google | High | Already the precedent in this platform's reference repos (`rook-server`'s per-service Firestore namespaces, `sight`'s `NoteService`/`SyncNotes`); Firebase's own documentation explicitly recommends this exact shape — "in a chat application, you might organize users and messages as collections nested within chat room documents" (see Web Citations) |
| Cloud SQL / managed Postgres | Alternative storage | Managed GCP service | Mature | Actively maintained | Low | No relational/join need — conversations and their messages are a pure parent/child access pattern; would break from the GCP-managed-document-store precedent already used elsewhere for no benefit |
| Self-hosted Postgres/SQLite on GKE | Alternative storage | OSS | Mature | Self-maintained | Lowest | Reintroduces operational burden (backups, scaling, patching) this platform has deliberately avoided everywhere else via managed services (constitution Principle V) |
| Sight's `internal/rpc`-style bounded wire-type package (in-house pattern, not a third-party library) | Shared client-library boundary for the two chat clients | N/A | Proven in a real, working repo (`sight`) | Actively used in that repo | High | Directly matches Q14's requirement (owner's TUI view and `bateau`, the guest's standalone client, must share protocol logic, not duplicate it). Adapted as a **public** package published from `rebus` (not an `internal/` package) since `rebus`, `bateau`, and `kingfish` are three separate repos/modules — see Architecture Patterns. |

**Selected:** Cloud Firestore for storage, with a
`conversations/{conversationId}/messages/{messageId}` subcollection
schema; a single shared Go package for the client-side wire types and
RPC calls, imported by both clients, following sight's bounded-import
pattern.
**Rejected (with reason):** Cloud SQL/Postgres and self-hosted SQL —
both rejected for the reasons in the table above; no relational need, and
self-hosting reintroduces avoided operational burden.
**Open questions:** none — see Decision Log.

## Architecture Patterns

- **Storage schema: `conversations/{conversationId}` parent documents,
  each with a `messages` subcollection.** Conversation document holds the
  two participant identities (admin's identity + the invited
  participant's, keyed by their registered public key) and `createdAt`.
  Each message document holds `sender`, `content`, `sentAt`, `updatedAt`
  (separate from `sentAt` — needed so an admin edit is distinguishable
  from a new message in sync queries), and `hiddenFor` (a per-participant
  visibility flag set when *the participant* "deletes" their own view,
  per the asymmetric-deletion decision — the admin's hard-delete instead
  physically removes the document). This is the structure Firebase's own
  documentation names as the standard pattern for exactly this kind of
  app (see Web Citations).
- **Incremental sync via `updatedAt` queries**, directly extending
  sight's `NoteService.SyncNotes(ctx, customerID, since time.Time)`
  pattern already cited in [002](002-hybrid-tui-layer.md): each client
  pulls `messages.where("updatedAt", ">", lastSyncTime)` per conversation
  on manual check (per the confirmed manual-check-only sync model).
  Firestore supports this directly; combining it with an equality filter
  on `hiddenFor` (to exclude messages a participant has hidden from their
  own sync) needs a composite index, which Firestore supports — single-
  inequality-field limit only restricts *inequality* filters, not
  equality ones (see Web Citations).
- **Clients never access Firestore directly — all access is mediated by
  the backend RPC API.** This is a direct consequence of the already-
  decided stateless RPC pattern: Firestore security rules are not the
  access-control mechanism here (the backend service account has full
  access); admin-vs-participant permission enforcement (who can hard-
  delete, who can edit) lives in the backend's application logic, not in
  Firestore rules.
- **Shared client-library boundary, sight-style — adapted for a
  cross-repo split.** A single Go package holds the wire/RPC types and
  the client-side calls (auth-challenge signing per rook-cli's pattern,
  conversation/message CRUD, incremental sync). Both the owner's
  TUI-shell-plugged-in chat view and the guest's separate, lighter
  standalone binary import *only* this package — never the backend's
  other internals directly — directly extending sight's own enforced
  rule that "`sight-cli` imports only types from `internal/rpc`... never
  from `internal/app`, `internal/store`, or `internal/service`" (see Web
  Citations). **One adaptation is required**: sight keeps `sightd` and
  `sight-cli` in one repo/module, so `internal/rpc` (Go's
  compile-time-enforced internal-package boundary) works as-is. This
  app's backend (`rebus`, `github.com/monamaret/rebus`) and its guest
  client (`bateau`, `github.com/monamaret/bateau`) are **separate
  repositories** (Decision Log) — and the owner's embedded view, living
  in `kingfish`'s own repo *(amended 2026-06-24, see
  [022-tui-shell-repo-boundary.md → Decision Log](022-tui-shell-repo-boundary.md#decision-log):
  originally written as "`skipper`'s own repo" — the shell and its
  embedded-view adapters live in `kingfish`, a separate repo from
  `skipper`, not in `skipper` itself)*, is a *third* separate module
  needing the same import. An `internal/` package is invisible outside
  its own module, so it cannot serve this role across three repos. The
  shared package is therefore **public** — published from `rebus` (e.g.
  `github.com/monamaret/rebus/client`, not under an `internal/` path) —
  and both `bateau` and `kingfish`'s embedded view import it as an
  external Go module dependency. The *discipline* sight enforces (clients
  see wire types and RPC calls only, never server internals) is
  preserved by convention and by what the package itself exposes, not by
  Go's compiler — the boundary moves from "enforced by `internal/`" to
  "enforced by what `rebus` chooses to export," the same tradeoff any
  published SDK package makes.
- **Local cache per client, rook-style.** Each client (owner's and
  guest's) keeps a local flat-file cache of synced messages per
  conversation, mirroring rook-cli's `stash/<space-id>/` pattern, so
  conversation history is browsable between manual syncs without a live
  connection.

## Existing Codebase Evaluation

- **`sight`** — `internal/rpc` package boundary (wire types only, no
  server-internals leakage to clients) is the direct model for this
  doc's shared-client-library decision; `NoteService.SyncNotes(since)` is
  the direct model for the incremental-sync query shape.
- **`rook-reference`** (`rook-cli`) — local flat-file cache pattern
  (`stash/<space-id>/`) is the model for each client's local message
  cache; the SSH-private-key-signs-an-auth-challenge-locally pattern is
  the model for both clients' auth flow (per the already-confirmed
  provisioning decision in [002](002-hybrid-tui-layer.md)).
- No `skipper` chat code exists yet — both clients and the backend are
  greenfield, building on these two repos' patterns.

## Security & Compliance

- **Confidentiality model (Q1, confirmed):** transport encryption (TLS) +
  server-side encryption-at-rest — not end-to-end encrypted. The backend
  is technically capable of accessing plaintext if needed. This is
  internally consistent with the admin-asymmetric capabilities confirmed
  under Message Lifecycle below (true E2EE would make server-side
  admin-delete and admin-edit-after-sending impossible). Q2/Q3 (E2EE key
  derivation, lost-key recovery) are now moot — N/A given this answer.
- Q15 (external compliance/legal retention constraint) — resolved below,
  see Compliance (Q15, confirmed) under Security & Compliance: no external
  regime applies.

### Message lifecycle (Q4–Q6, confirmed)

- **Retention:** indefinite — no automatic expiry/rolling window.
- **Deletion is asymmetric, not peer-to-peer:**
  - The **admin/owner can truly delete** a message — removes the
    server-side log entry.
  - The **participant can also delete a message from their own view**,
    but if it was already synced to the server, their deletion does
    **not** remove the server-side copy — the admin's log retains it
    regardless. From the participant's perspective this is closer to
    "hide for me" than a true delete.
- **Editing is admin-only:** the admin/owner can edit a message after
  sending; the participant cannot edit their own sent messages.
- **Export:** the owner can export/back up the full message history
  locally (Markdown/JSON, stash-app-style) — but this is **off by
  default**, an explicit opt-in action, not automatic.

## Performance, Reliability & Operations

- **Delivery model (Q7, confirmed): strictly asynchronous store-and-forward**
  — messages queue on the backend and are pulled on next sync (rook's
  pull-sync model). No live/persistent-connection mode. Q8 (polling vs.
  persistent connection) is N/A given this answer.
- **Sync/notification UX (Q9 + Q12, confirmed): manual check command
  only** — no background polling. The user explicitly runs a
  check/refresh action to pull new messages; there is no ambient
  "arrival latency" to target beyond "whenever you next check." This also
  resolves Q9 (the old latency-target question) by replacing it with "no
  target — sync is on-demand."
- **Unread-state UI (Q13): deferred, not part of this interview.** The
  owner clarified the app-catalog (top-level platform launcher) sits
  several screens above where in-conversation notification/unread state
  would actually be designed — that's an in-app UI design question for
  the chat app itself, to be worked out during its own design phase, not
  resolved here.
- **Follow-up flagged: chat notifications on the global status bar.**
  The owner has since asked that chat notifications surface on the
  shell's global status bar (not just an in-app indicator), and that this
  become a convention other apps with notifications can reuse — not a
  chat-specific mechanism. This is a *shell-level* concern, tracked in
  [007-tui-shell-dispatch-and-catalog.md → Open Questions](007-tui-shell-dispatch-and-catalog.md#open-questions)
  (generic cross-app notification convention, plus the open question of
  what happens to status-bar notifications during an SSH-passthrough
  session). Not resolved here — Q13's in-app design and this shell-level
  convention are two different, related follow-ups.
- **Follow-up flagged: non-manual (lazy/configurable) sync.** Chat's
  manual-check-only sync (Q7/Q12, Decision Log) is the confirmed v1
  default; whether a lazy or user-configurable background sync mode is
  worth adding later is tracked as a shared follow-up alongside the
  catalog's identical default — see
  [007 → Open Questions](007-tui-shell-dispatch-and-catalog.md#open-questions).

### Conversation & identity scope (Q10–Q11, confirmed)

- **Multiple separate 1:1 conversations** — the owner can invite and
  maintain several different participants, each in their own distinct
  thread, not just one fixed relationship. This means the TUI/catalog
  needs a conversation list, not a single chat entry.
- **Single-device per identity for v1** — each participant identity maps
  to exactly one registered key; multi-device support is deferred, not
  built now.

### Outside-domain participant client (Q14, confirmed)

The invited outside-domain participant uses a **lighter-weight,
standalone chat-only client**, not the full `kingfish` TUI
shell — the rest of the platform (catalog, other apps) is irrelevant to a
guest who isn't using anything else. This is a separate, minimal client
built from the same backend API the owner's TUI uses, not a cut-down
build of the full shell.

### Compliance (Q15, confirmed)

No external compliance/legal retention regime applies — this is purely a
private, personal security posture, not something with an outside
constraint to satisfy.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Designing storage/transport before the confidentiality model is settled, then having to redo it for E2EE | Medium | High | Avoided — the interview was answered before any Libraries & Stack/Architecture Patterns work started | mona |
| Treating "secure" as satisfied by transport encryption (TLS) alone when the owner actually expects end-to-end encryption | Medium | High | Resolved directly — Q1 confirmed transport+at-rest is the intended model, not an assumed default | mona |
| Building a second, separate guest client (Q14) duplicates logic already in the owner's TUI client | Medium | Low–Medium | Both clients should be thin wrappers over the same backend API (per the stateless pattern already chosen) — share the client-side library/SDK between them rather than duplicating protocol logic | mona |

## Open Questions

All 15 interview questions are answered (Q2/Q3/Q8 became N/A as a
consequence of other answers; Q13's in-app unread-state UI is explicitly
deferred to the chat app's own design phase, not a blocker here). The
comparative research (storage schema, sync mechanics, shared-client-
library approach) is also done — see Architecture Patterns and Decision
Log. This doesn't block updating
[specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md)'s
Scope with these concrete specifics. Four follow-up items were originally
flagged for future research (not v1-blocking) — all four are now resolved
by sibling docs:

- [x] ~~SSH session-state UX for the chat app~~ — **Resolved by
  [014-guest-chat-client-scope-and-auth-ux.md](014-guest-chat-client-scope-and-auth-ux.md):**
  SSH agent first (if `SSH_AUTH_SOCK` is set), falling back to reading
  the private key file directly with a passphrase prompt if encrypted; a
  missing key exits with a clear, actionable error.
- [x] ~~Is SSH the right auth mechanism for this app specifically?~~ —
  **Resolved by [014](014-guest-chat-client-scope-and-auth-ux.md): yes,
  SSH-key auth remains correct for the guest too** — the realistic guest
  profile makes key setup low-friction, and every alternative considered
  either breaks the platform's one-identity-system principle or loses
  durable, revocable identity. The fix needed was a smoother bootstrap,
  not a different mechanism.
- [x] ~~Full feature scope of the minimal chat-only TUI app for invited
  guests~~ — **Resolved by [014](014-guest-chat-client-scope-and-auth-ux.md):**
  GitHub Releases (`goreleaser`) + `go install` distribution (biblio's
  precedent); an interactive TUI plus `history`/`send` non-interactive
  commands (bbb-le's CLI shape); XDG config storage (rook-cli's
  convention).
- [x] ~~Stash's sync/stash commands and backend data model need their own
  detailed-design pass, the way this doc did for chat.~~ — **Resolved by
  [009-markdown-stash-application.md](009-markdown-stash-application.md):**
  Cloud Storage for file content + Firestore for metadata (Firestore's
  1 MiB per-document cap rules out Firestore-only storage), the same
  `updatedAt`-incremental-sync pattern reused here from chat, soft delete
  with a recovery window, overwrite-in-place (no version history), and
  folder/tag organization. The backend's repo was separately named
  `pocket` by [018](018-stash-backend-repo-naming-and-boundary.md).

## Decision Log

- **2026-06-24** — **The chat application's two components are named and
  will live in their own separate repositories, neither inside
  `skipper`'s own repo:** the backend (server/Kubernetes side) is
  **`rebus`**, at `github.com/monamaret/rebus` (not yet created); the
  guest's standalone thin client is **`bateau`**, at
  `github.com/monamaret/bateau` (not yet created). Full implementation/
  code/reference/spec for the backend lives in `rebus`'s own repo; full
  implementation/code/reference/spec for the guest client lives in
  `bateau`'s own repo — `skipper`'s own repo holds only the scaffold/
  deployment configuration that targets them (the persisted Kustomize
  base at `deployments/rebus/`, the catalog entry, the owner's
  TUI-embedded view that imports `rebus`'s published client package).
  This is the first concrete instance of constitution Principle VIII's
  repo-separation requirement actually being realized with real names.
  — Decided by: mona.
- **2026-06-24** — **Confidentiality model: transport (TLS) + at-rest
  encryption only, not end-to-end encrypted.** The backend can technically
  access plaintext if needed. — Rationale: consistent with the admin's
  elevated capabilities (server-side delete, edit-after-sending) decided
  below, which a true E2EE design would make impossible. — Decided by:
  mona.
- **2026-06-24** — **Message retention is indefinite — no automatic
  expiry.** — Decided by: mona.
- **2026-06-24** — **Deletion is asymmetric: the admin/owner can truly
  delete a message (removes the server-side log); the participant can
  delete from their own view, but that does not remove the server-side
  copy once synced.** — Rationale: the admin retains an authoritative,
  immutable log regardless of what the participant does on their end —
  a deliberate trust asymmetry, not an oversight. — Decided by: mona.
- **2026-06-24** — **Editing after sending is admin-only — the
  participant cannot edit their own sent messages.** — Decided by: mona.
- **2026-06-24** — **Local export/backup of message history is
  supported but off by default** (explicit opt-in action, not automatic,
  stash-app-style). — Decided by: mona.
- **2026-06-24** — **Delivery is strictly asynchronous store-and-forward**
  — no live/persistent-connection mode. — Rationale: consistent with the
  stateless rook/sight pattern already chosen; simplest path, no
  always-open connection to maintain. — Decided by: mona.
- **2026-06-24** — **The owner can hold multiple separate 1:1
  conversations with different invited participants** (still never group
  chats) — not one single fixed relationship. — Rationale: matches a
  general-purpose private chat capability rather than over-narrowing
  scope to one relationship that can't be reused for future contacts. —
  Decided by: mona.
- **2026-06-24** — **Single device/key per participant identity for
  v1** — multi-device support is deferred. — Rationale: matches v1's
  minimal-scope bias (Principle I); revisit only if a real need for a
  second device shows up. — Decided by: mona.
- **2026-06-24** — **Sync is manual-check-only — no background polling.**
  — Rationale: matches the async-only delivery model and the
  no-live-connection constraint; simplest, fits the stateless pattern.
  Replaces the original "latency target" question with "no target — sync
  is on-demand." — Decided by: mona.
- **2026-06-24** — **In-app unread-state UI design (Q13) is deferred to
  the chat app's own UI design phase** — not resolved as part of this
  discovery interview. — Rationale: the owner clarified this is several
  screens below the top-level platform catalog and needs its own design
  pass once the chat app's screens are actually being built. — Decided
  by: mona.
- **2026-06-24** — **The outside-domain participant uses a lighter-weight,
  standalone chat-only client, not the full `kingfish` TUI shell.** —
  Rationale: the rest of the platform (catalog, other apps) is irrelevant
  to a guest using only chat; both clients should share the underlying
  client-side library/SDK over the same backend API rather than
  duplicating protocol logic. — Decided by: mona.
- **2026-06-24** — **No external compliance/legal retention regime
  applies.** — Rationale: purely a private, personal security posture. —
  Decided by: mona.
- **2026-06-24** — **Storage: Cloud Firestore, with a
  `conversations/{conversationId}/messages/{messageId}` subcollection
  schema, queried incrementally via an `updatedAt` field. Clients never
  access Firestore directly — only via the backend's RPC API.** —
  Rationale: matches Firebase's own documented recommendation for this
  exact app shape, extends sight's already-cited `SyncNotes(since)`
  incremental-sync pattern, and keeps access control in application code
  rather than relying on Firestore security rules (consistent with the
  already-decided stateless-RPC architecture). Rejected Cloud SQL/
  self-hosted Postgres — no relational need, and self-hosting reintroduces
  operational burden the platform avoids elsewhere. — Decided by: mona.
- **2026-06-24** — **Both chat clients (owner's TUI view, guest's
  standalone client) share one Go package for wire types and RPC calls —
  neither imports the backend's internal packages directly.** —
  Rationale: directly reuses sight's enforced import-boundary pattern
  (`sight-cli` imports only `internal/rpc` types), which is exactly the
  mechanism needed to satisfy Q14's requirement that the two clients not
  duplicate or drift apart on protocol logic. — Decided by: mona.

## Internal References

- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — confirmed chat
  architecture (interaction model, protocol, provisioning) this doc does
  not reopen
- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — where
  chat's eventual web client would live if ever built
- [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md) —
  the feature item this research will refine with concrete specifics
- [014-guest-chat-client-scope-and-auth-ux.md](014-guest-chat-client-scope-and-auth-ux.md) —
  `bateau`'s full command/distribution/auth design
- `github.com/monamaret/rebus` (not yet created) — the chat backend's own repo, named in this doc's Decision Log
- `github.com/monamaret/bateau` (not yet created) — the guest client's own repo, named in this doc's Decision Log
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) —
  Principle VI (secure-by-default), Principle VIII (repo separation — this
  decision's first concrete instance), and Security & Operational
  Posture, directly relevant once the confidentiality model is chosen

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Cloud Firestore — Structure Data | https://firebase.google.com/docs/firestore/manage-data/structure-data | 2026-06-24 | Confirms the subcollection pattern this doc's schema uses; explicitly names "a chat application... organize users and messages as collections nested within chat room documents" as the recommended shape for this exact use case. |
| Cloud Firestore — Query Data | https://firebase.google.com/docs/firestore/query-data/queries | 2026-06-24 | Confirms greater-than timestamp queries for incremental sync, and the single-inequality-field constraint (equality filters, like the `hiddenFor` filter this doc's sync query needs, don't count against that limit). |
| `github.com/monamaret/sight` — `.specify/memory/constitution.md` (Additional Constraints) | https://github.com/monamaret/sight | 2026-06-24 (fetched earlier in this research thread) | Source of the "`sight-cli` imports only types from `internal/rpc`... never from `internal/app`, `internal/store`, or `internal/service`" boundary rule this doc's shared-client-library decision directly reuses. |
| `github.com/monamaret/sight` — `internal/domain/note_service.go` | https://github.com/monamaret/sight | 2026-06-24 (fetched earlier in this research thread) | Source of the `SyncNotes(ctx, customerID, since time.Time)` incremental-sync signature this doc's `updatedAt`-query pattern extends. |
| `github.com/monamaret/rook-reference` — `specs/architecture/component-overview.md` | https://github.com/monamaret/rook-reference | 2026-06-24 (fetched earlier in this research thread) | Source of the local flat-file cache (`stash/<space-id>/`) and SSH-key-signs-auth-challenge-locally patterns this doc's client-cache and auth-flow recommendations reuse. |

## Appendix

### Discovery Interview

Answer these to unblock the rest of this research doc. Organized by
topic; answers will be transcribed into the Decision Log above and used
to update [005-private-chat-1to1.md](../features/005-private-chat-1to1.md)'s
Scope with concrete specifics.

**A. Security & confidentiality model** — ✅ resolved, see Decision Log

1. ~~End-to-end encrypted vs. transport+at-rest only?~~ — **Transport
   (TLS) + at-rest encryption only.** The backend can technically access
   plaintext if needed.
2. ~~If E2EE: key derivation?~~ — **N/A**, given Q1.
3. ~~If E2EE: lost-key recovery?~~ — **N/A**, given Q1.

**B. Message lifecycle & storage** — ✅ resolved, see Decision Log

4. ~~Retention period?~~ — **Indefinite.**
5. ~~Delete/edit semantics?~~ — **Asymmetric.** Admin/owner can truly
   delete (removes the server-side log) and can edit after sending.
   Participant can delete from their own view only — does not remove the
   server-side copy once synced — and cannot edit.
6. ~~Export/backup?~~ — **Yes, supported, off by default** (explicit
   opt-in action).

**C. Delivery & transport** — ✅ resolved, see Decision Log

7. ~~Async-only vs. also live mode?~~ — **Strictly asynchronous
   store-and-forward** (pull-sync, rook-style). No live mode.
8. ~~If live: polling vs. persistent connection?~~ — **N/A**, given Q7.
9. ~~Latency target?~~ — **Folds into Q12** (sync frequency = effective
   latency, since there's no live mode).

**D. Identity, scope & multi-conversation** — ✅ resolved, see Decision Log

10. ~~One fixed relationship vs. multiple conversations?~~ — **Multiple
    separate 1:1 conversations**, each with a different invited
    participant (still never group chats).
11. ~~Multi-device?~~ — **Single device/key per identity for v1.**

**E. Notifications & TUI UX** — ✅ resolved (Q13 explicitly deferred), see Decision Log

12. ~~Notification/sync experience?~~ — **Manual check command only** —
    no background polling.
13. ~~Unread state in catalog vs. in-app?~~ — **Deferred** — the owner
    clarified this is an in-app design question for the chat app's own
    screens, several levels below the top-level catalog; not resolved as
    part of this interview.

**F. Outside-domain participant experience** — ✅ resolved, see Decision Log

14. ~~Full TUI shell vs. lighter client?~~ — **Lighter-weight, standalone
    chat-only client** for the guest — not the full `kingfish` TUI shell.

**G. Compliance / non-functional** — ✅ resolved, see Decision Log

15. ~~External compliance constraint?~~ — **None** — purely a private,
    personal security posture.
