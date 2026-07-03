# Research: Markdown stash application — detailed design

**Status:** Complete — all open questions resolved; ready to update specs/features/004-markdown-stash-sync.md with concrete specifics
**Date:** 2026-06-24
**Owner:** mona

**Naming note (added 2026-06-24):** [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md)
named this doc's backend **`pocket`**, at `github.com/monamaret/pocket`
(not yet created) — its own repo, per constitution Principle VIII, not
inside `skipper`'s. `skipper`'s repo holds only the deployment config and
catalog entry targeting it, plus the owner's embedded-view adapter
importing `pocket`'s published client package.

## Problem Statement

[002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) settled category 1
(dev-artifact/context work) at the architecture-pattern level: a local
Markdown stash, rook pattern, source-controlled-style `.md` files,
pull-synced — explicitly *not* a context-engine/entity-search service.
[specs/features/004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md)
already carries that pattern into a feature item, but — as flagged
explicitly in [006-private-chat-application.md → Open Questions](006-private-chat-application.md#open-questions)
— stash has never had the same detailed-design pass chat got in `006`:
concrete backend data model, delete/versioning semantics, and
organizational structure. This doc closes that gap.

- **Goal:** A concrete backend data model and command-semantics design
  for the stash app, narrow enough to update `004`'s Scope with real
  specifics instead of placeholders — mirroring what `006` did for chat.
- **In scope:** Storage split between metadata and file content, local
  cache shape, delete/versioning semantics, organizational structure
  (flat vs. folders/tags), sync mechanics.
- **Out of scope:** Re-opening `002`'s settled pattern (local Markdown
  stash, not a context-engine service) — not reconsidered here.
- **Success criteria:** Every open item resolved (directly, or via owner
  decision) and recorded in this doc's Decision Log, then
  `004-markdown-stash-sync.md` updated to match.
- **Non-goals:** Building or comparing alternative sync libraries before
  this design is settled — `002` already chose the architecture pattern
  (sight's `SyncNotes(since)` shape); this doc fills in the data model
  and semantics around that already-chosen pattern, not a new survey.

## Background & Context

- **Relevant prior work:**
  - [002 → Architecture Patterns, category 1](002-hybrid-tui-layer.md#architecture-patterns) — local Markdown stash (rook pattern), source-controlled-style files, pull-synced; context-engine service explicitly rejected.
  - [002 → Libraries & Stack](002-hybrid-tui-layer.md#libraries--stack) — rook-cli's local flat-file store (`stash/<space-id>/`) and sight's `NoteService.SyncNotes(ctx, customerID, since time.Time)` incremental-sync shape, both already named as the concrete mechanisms to follow.
  - [004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md) — the feature item this doc will refine; already flags the storage-split question as open.
  - [007-tui-shell-dispatch-and-catalog.md → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log) — stash's catalog entry type (`embedded-view`, `viewID: stash`), the shared UX convention it honors, and the platform-wide manual-sync-only default it already follows.
- **Domain assumption:** Unlike chat, stash has no multi-party trust
  model — it's the owner's own content, synced between their own
  devices/sessions. This rules out a whole category of design questions
  chat needed (no admin/participant asymmetry, no outside-domain
  provisioning) and makes this doc's design space narrower than `006`'s.
- **Stakeholders:** mona (sole owner/operator and sole stash user).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **Cloud Storage** (content) + **Firestore** (metadata/index) | Stash file storage, split by concern | Proprietary GCP managed services | Mature | Actively maintained by Google | High | Resolves `004`'s open storage-split question — see Architecture Patterns for rationale. |
| Firestore alone (content stored as a document field) | Single-service storage | Proprietary GCP managed service | Mature | Actively maintained | Rejected | Firestore documents are capped at 1 MiB; arbitrary Markdown files (especially ones with embedded content/length over time) risk exceeding that, where chat's short messages did not — see Web Citations for the documented limit. |
| Self-hosted object storage (e.g. MinIO on GKE) | Alternative to Cloud Storage | OSS | Mature | Self-maintained | Rejected | Reintroduces the same self-hosting operational burden this platform has rejected everywhere else (constitution Principle V) for no benefit over the managed equivalent. |

**Selected:** Cloud Storage for file content, Firestore for metadata/index
— see Architecture Patterns for the concrete schema.
**Rejected (with reason):** Firestore-only storage (document size limit);
self-hosted object storage (unjustified operational burden). See table.
**Open questions:** none for this section — resolved directly from the
size-limit constraint, not an owner preference call.

## Architecture Patterns

- **Storage split: Cloud Storage for content, Firestore for metadata.**
  Each stashed file's Markdown content lives as a Cloud Storage object
  (keyed by a stable ID); a corresponding Firestore document holds
  `filename`, `path` (folder grouping), `tags` (list, optional),
  `size`, `createdAt`, `updatedAt`, `deletedAt` (nullable — soft delete,
  see Decision Log), and the Cloud Storage object key. This resolves
  `004`'s open storage-split question directly — Firestore's
  per-document size cap (see Web Citations) makes Cloud-Storage-for-
  content the only safe choice once file sizes aren't bounded the way
  chat's short messages were.
- **Incremental sync** follows the same `updatedAt`-field query pattern
  already established for chat in `006` (`files.where("updatedAt", ">",
  lastSyncTime)`), directly extending sight's `SyncNotes(since)` shape
  `002` already named — no new sync mechanism, the same one reused a
  second time. Sync queries exclude soft-deleted files past their
  recovery window via a `deletedAt`-aware filter, mirroring chat's
  `hiddenFor` equality-filter pattern from `006`.
- **Soft-delete purge** is a separate, deferred mechanism (a periodic or
  manual cleanup of `deletedAt`-past-window documents/objects) — not
  designed in detail here; the recovery-window *length* and the purge
  *trigger* (manual command vs. a scheduled job) are implementation
  details to settle during planning, not research blockers.
- **Folder/tag organization** is metadata-only (the `path`/`tags`
  fields above) — it does not change the storage split or sync
  mechanism, just what the TUI view groups/filters by when listing
  stashed files.
- **Local cache mirrors rook-cli's `stash/<space-id>/` shape exactly**,
  minus the `space-id` segment: since this platform has no multi-tenant/
  multi-space concept (unlike rook's team-oriented design), the local
  cache is a single flat `stash/` directory of `.md` files, each with a
  `.json` sidecar carrying the same metadata Firestore holds (`updatedAt`,
  Cloud Storage key) — enabling fully offline browsing between syncs,
  per the already-established cache-with-fallback discipline used
  elsewhere on this platform (the app catalog, rook-cli itself).
- **No "spaces" or multi-tenant namespacing** — a deliberate
  simplification from rook's own design, since rook's `space-id` exists
  for multi-user/team contexts this platform doesn't have (Principle I:
  don't carry over multi-tenant design weight from a reference repo built
  for a different scale).

**Tradeoffs:** Splitting storage across two GCP services (vs. one) is a
small amount of additional integration work (keeping the Firestore
metadata doc and the Cloud Storage object in sync on every write) in
exchange for not hitting Firestore's document-size ceiling as content
grows — a clear win given the actual constraint, not a stylistic choice.
**Integration points:** the same `updatedAt`-query sync pattern is now
used by both chat (`006`) and stash — worth factoring into a shared
backend helper if a third app ever needs the same incremental-sync shape,
per `002`'s own "Integration points" note about reusing caching
discipline rather than re-implementing it per app.
**Data model implications:** this is the first concrete data model
written for stash — previously only described as "Firestore for
metadata, with file content alongside or in Cloud Storage" (the
open question this doc now resolves) in `004`.

## Existing Codebase Evaluation

No `skipper` stash implementation exists yet. `rook-cli`'s
`stash/<space-id>/` local-cache shape and sight's `SyncNotes(since)`
incremental-sync signature remain the direct precedents, as already
named in `002`.

## Security & Compliance

- Stash content is the owner's own personal notes/artifacts — no
  multi-party confidentiality model is needed (unlike chat's
  transport-vs-E2E question), since there's only one party with access
  by construction (no sharing/invite mechanism for stash, unlike chat's
  outside-domain participant).
- Standard GCP-managed encryption at rest (Cloud Storage, Firestore) and
  in transit (TLS) applies, consistent with constitution Principle VI —
  no additional encryption requirement beyond what's already the
  platform default.
- No new credential type — auth follows the same SSH-key-signed-challenge
  pattern already established platform-wide (rook-cli's pattern, per
  `002`).

## Performance, Reliability & Operations

- Sync is manual-check-only (no background polling), consistent with
  `007`'s platform-wide default — no new performance characteristic to
  design around beyond what that decision already covers.
- Cloud Storage + Firestore's combined read/write path for a single
  stash operation (one object write, one document write) is well within
  normal latency expectations for a manual, user-triggered action — not
  a performance-sensitive path requiring special design.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Firestore metadata doc and Cloud Storage object drift out of sync (e.g. a failed write leaves one updated and not the other) | Low–Medium | Medium | Write the Cloud Storage object first, then the Firestore metadata doc referencing it — an orphaned object with no metadata is harmless (just unreferenced), while the reverse (metadata pointing at a missing object) would be a real bug; ordering the writes this way avoids the worse failure mode | mona |
| Carrying over rook's `space-id` concept speculatively, before any multi-space need exists | Low (already avoided by this doc's decision) | Low | Explicitly decided against above — flat namespace for v1, revisit only if a real multi-space need shows up | mona |

## Open Questions

All three questions originally listed here are resolved — see the
Decision Log.

- [x] ~~Delete semantics?~~ — **Soft delete with a recovery window.**
- [x] ~~Versioning?~~ — **Overwrite in place, no history.**
- [x] ~~Organizational structure?~~ — **Folder/tag organization**, not a
  flat list only.

## Decision Log

- **2026-06-24** — **Deleting a stashed file is a soft delete**: the
  Firestore metadata doc is marked deleted (e.g. a `deletedAt` field) and
  the Cloud Storage object is retained for some recovery window before
  permanent purge, rather than removing both immediately. — Rationale:
  a safety net against accidental deletes, reasonable given this is the
  owner's own working notes/artifacts where an accidental hard delete
  would be costly to lose; the extra state (a `deletedAt` field, an
  eventual purge step) is modest compared to the cost of an
  unrecoverable mistake. — Decided by: mona.
- **2026-06-24** — **Re-stashing/editing a file overwrites it in place —
  no version history is retained.** — Rationale: these are personal
  notes synced like rook's stash, not a version-controlled document
  system; if history matters for a given file, that's what source
  control (git) is for, not this app (Principle I — don't build
  speculative versioning machinery for a need source control already
  covers). — Decided by: mona.
- **2026-06-24** — **Stash supports folder/tag-based organization, not
  just a flat list.** The Firestore metadata schema gains a `path` (for
  folder-style grouping) and/or `tags` field alongside the fields already
  decided above. — Rationale: the owner's actual usage pattern needs
  grouping beyond a flat list, even though biblio's own flat-directory
  convention was the simpler default to reach for — that convention was
  for *site-type* content layout, not a personal stash's own internal
  organization, so it doesn't directly transfer here. — Decided by: mona.

## Internal References

- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — category-1 architecture pattern this doc implements concretely
- [004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md) — the feature item this research will refine
- [006-private-chat-application.md](006-private-chat-application.md) — the sync-pattern precedent (`updatedAt` incremental queries) reused here, and the source of the follow-up that prompted this doc
- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) — catalog entry type and UX convention this app honors
- [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) — names this doc's backend `pocket` and its own repo
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and Principle VI (secure-by-default), both directly relevant above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Cloud Firestore — Quotas and limits | https://firebase.google.com/docs/firestore/quotas | 2026-06-24 | Confirms Firestore's maximum document size (1 MiB), the concrete constraint that rules out storing arbitrary Markdown file content directly in a Firestore document and justifies the Cloud Storage + Firestore split. |

## Appendix

None yet.
