# Research: Stash backend repo naming and the skipper/implementation boundary

**Status:** Complete — repo named (`pocket`), boundary resolved; ready to update specs/research/009-markdown-stash-application.md and specs/features/004-markdown-stash-sync.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[009-markdown-stash-application.md](009-markdown-stash-application.md)
designed stash's complete backend data model (Cloud Storage + Firestore,
soft delete, folder/tag organization) and
[004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md)
scopes its deployment via `skipper app add --github <repo>` — but
neither has ever named *which* repo, or confirmed that stash's backend
implementation lives outside `skipper`'s own repo at all. This is the
exact gap [006-private-chat-application.md](006-private-chat-application.md)/
[013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md)
closed for chat (naming `rebus`/`bateau`, each in their own repo) —
stash never got the equivalent treatment.

- **Goal:** Confirm stash's backend lives in its own, separately-named
  repository (per constitution Principle VIII), establish the same
  skipper/implementation boundary already established for chat, and name
  that repo.
- **In scope:** The repo-separation boundary for stash specifically
  (what lives in `skipper`'s repo vs. the new one); whether stash needs
  more than one new repo (chat needed two — backend + guest client);
  the repo's name.
- **Out of scope:** Re-opening `009`'s settled data-model decisions
  (storage split, soft delete, organization) — not reconsidered here.
- **Success criteria:** A named repo and a confirmed skipper/
  implementation boundary, recorded in this doc's Decision Log.
- **Non-goals:** Designing the stash backend's actual internal code
  structure beyond what `009` already specified — this doc is about repo
  boundaries and naming, not new technical design.

## Background & Context

- **Relevant prior work:**
  - [006 → Decision Log](006-private-chat-application.md#decision-log) — named `rebus` (backend) and `bateau` (guest client), each in its own repo, as the first concrete instance of Principle VIII's repo-separation rule.
  - [013 → Architecture Patterns](013-cli-command-mapping-and-privilege-model.md#architecture-patterns) — the established convention this doc applies to stash: `skipper` holds only deployment config (a persisted Kustomize base) and any embedded-view adapter that imports the backend's published client package; the backend's own implementation, schema, and any client package it publishes live in the backend's own repo.
  - [009-markdown-stash-application.md](009-markdown-stash-application.md) — the complete data model/semantics this doc's named repo will implement.
  - [.specify/memory/constitution.md Principle VIII](../../.specify/memory/constitution.md) — "individual web applications, separately-deployed application binaries, and other projects `skipper` scaffolds or builds... live and are maintained in their own separate repositories... never inside `skipper` itself."
- **Domain assumption:** Unlike chat, stash has **no second client** to
  name — per `009`'s own domain assumption, stash has no multi-party
  trust model (no admin/participant asymmetry, no outside-domain
  provisioning), so there is no "guest" equivalent needing its own repo
  the way `bateau` does for chat. Stash needs exactly **one** new repo:
  the backend.
- **Stakeholders:** mona (sole owner/operator, and sole stash user).

## Libraries & Stack

Not applicable — this doc is a naming/boundary decision, not a
technology comparison. No new library or service candidate is evaluated
here.

## Architecture Patterns

- **Stash's backend is named `pocket`, at `github.com/monamaret/pocket`
  (not yet created).** The naming itself was the owner's own creative
  call, the same way `rebus`/`bateau` were for chat — not something
  derived from research.
- **The skipper/implementation boundary, applied to stash exactly as
  established for chat in `013`:**
  - **`skipper`'s own repo holds:** the persisted Kustomize base at
    `deployments/pocket/` (per [010](010-gke-manifest-generation-and-upgrade.md#architecture-patterns))
    and the app catalog entry (`embedded-view`, `viewID: stash`, per
    [007](007-tui-shell-dispatch-and-catalog.md#decision-log)).
    *(Amended 2026-06-24, see
    [022-tui-shell-repo-boundary.md → Decision Log](022-tui-shell-repo-boundary.md#decision-log):
    the owner's TUI view itself — a Bubble Tea model plugged into the
    shell — does **not** live in `skipper`'s repo as originally written
    here; it lives in `kingfish`, the shell's own separate repo, which
    imports `pocket`'s published client package as an external Go module
    dependency.)*
  - **`pocket`'s own repo holds:** the backend service (the Firestore/
    Cloud Storage data model from `009`, the RPC/HTTPS API, the sync
    endpoint), and a **public** Go client package (e.g.
    `github.com/monamaret/pocket/client`, not an `internal/` package, for
    the same reason established in `006`'s Decision Log for `rebus`/
    `bateau` — the owner's embedded view lives in a different repo/module
    and needs an externally-importable package to depend on).
- **No second repo is needed for stash, unlike chat's two (`rebus` +
  `bateau`).** Stash's only client is the owner's own TUI view, which —
  same as chat's owner-side view — lives in `skipper`'s repo as a thin
  adapter, not a separately-distributed binary. There is no equivalent
  to `bateau` for stash, since there's no second party with their own
  standalone client to distribute.

**Tradeoffs:** None beyond what's already accepted for the analogous
chat decision — this is a direct application of an already-justified
pattern, not a new tradeoff.
**Integration points:** Reuses `010`'s Kustomize/persistence mechanism
and `007`'s catalog schema exactly as already designed for any
GitHub-repo-built backend app — no new deployment mechanism needed.
**Data model implications:** None new — `009`'s already-confirmed
Firestore/Cloud Storage schema simply now has a concrete repo to live in
once named.

## Existing Codebase Evaluation

No `pocket` code exists yet, and the repo itself hasn't been created yet.
`rebus` (chat's backend, also not yet created — see `006`) is the direct
structural precedent: same boundary, same "public client package, not
`internal/`" requirement, same Kustomize-deployment relationship with
`skipper`'s repo.

## Security & Compliance

No change from `009`'s already-confirmed posture (standard GCP-managed
encryption, SSH-key-signed-challenge auth, no new credential type) — this
doc is purely about repo boundaries, not security design.

## Performance, Reliability & Operations

Not applicable — no new runtime characteristic; this doc only concerns
where code lives, not how it runs.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Stash's implementation accidentally starts inside `skipper`'s own repo before `github.com/monamaret/pocket` actually exists | Low–Medium | Low | Don't begin coding `pocket`'s backend until that repo is actually created — `skipper app add --github github.com/monamaret/pocket` has nothing to point at until then | mona |

## Open Questions

None — resolved below.

## Decision Log

- **2026-06-24** — **The stash backend is named `pocket`, at
  `github.com/monamaret/pocket` (not yet created).** Full implementation/
  code/reference/spec for the stash backend lives in `pocket`'s own repo
  — `skipper`'s own repo holds only the deployment config and catalog
  entry that target it, plus the owner's embedded-view adapter importing
  `pocket`'s published client package. — Decided by: mona.
- **2026-06-24** — **Stash's backend lives in its own, separate
  repository — not inside `skipper`'s own repo** — applying constitution
  Principle VIII the same way `006`/`013` already did for chat. —
  Rationale: consistent with the already-established platform-wide
  repo-separation convention; no reason for stash to be an exception. —
  Decided by: mona.
- **2026-06-24** — **Stash needs exactly one new repo (the backend), not
  two like chat** — there is no second client/repo equivalent to
  `bateau`, since stash has no multi-party trust model and no second
  client to distribute. — Rationale: directly follows from `009`'s
  already-confirmed single-user domain assumption. — Decided by: mona.
- **2026-06-24** — **The skipper/implementation boundary mirrors `013`'s
  exactly**: `skipper`'s repo holds only the persisted Kustomize base,
  catalog entry, and the owner's embedded-view adapter; the new repo
  holds the backend and a public (not `internal/`) Go client package the
  adapter imports. — Decided by: mona.

## Internal References

- [009-markdown-stash-application.md](009-markdown-stash-application.md) — the data model this doc's named repo will implement
- [006-private-chat-application.md](006-private-chat-application.md) and [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) — the precedent (`rebus`/`bateau` naming and boundary) this doc applies to stash
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — the Kustomize deployment mechanism this doc's new repo deploys through
- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) — the catalog entry type (`embedded-view`) this doc's owner-side view registers as
- [specs/features/004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md) — the feature item this research will refine once the repo is named
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle VIII (repo separation), directly load-bearing above

## Web Citations

No external web research was needed — this doc applies an already-cited,
already-decided internal pattern (`006`/`013`) to a new case, not new
technical research.

## Appendix

None yet.
