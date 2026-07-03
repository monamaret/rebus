# Research: TUI shell repo boundary and naming (`kingfish`)

**Status:** Complete — repo named (`kingfish`), boundary resolved, scaffold-command shape decided; ready to amend `006`, `013`, `018`, `roadmap.md`, and `architecture.md` to correct the inconsistency this doc closes
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

Every other deployable thing in this platform has an explicit "what repo,
what's the skipper/implementation boundary" decision: `017` for standalone
content sites (wraps `biblio-cli`, owns nothing afterward), `018` for
stash's backend (`pocket`, new repo), `006`/`013` for chat (`rebus` +
`bateau`, two new repos). **The TUI shell itself never got this
treatment**, and the docs that exist are inconsistent about where it
lives:

- Constitution Principle VIII and `architecture.md`'s layer diagram are
  explicit: the hybrid TUI layer is "generated as its own repo per
  project; not vendored into `skipper`" — same treatment as the web layer.
- But `006`'s Architecture Patterns section says the owner's embedded chat
  view is "living in `skipper`'s own repo," and `roadmap.md`'s chat
  section repeats this ("living in `skipper`'s own repo"). `018` mirrors
  the same assumption for stash's owner-side adapter. `architecture.md`'s
  "First major release" diagram bundles the shell, the stash adapter, and
  the chat adapter into one box with no repo boundary drawn at all.

This is a real contradiction, not a stylistic inconsistency: the shell's
embedded-view drivers (`007`) resolve `viewID`s against a **compile-time**
registry (no dynamic plugin loading, per Principle I) — which means the
owner's stash/chat embedded views *must* be compiled into the same binary
as the shell itself. Wherever that binary's source lives is where those
adapters necessarily live too. So this question isn't cosmetic — it
determines which repo several already-designed features' code actually
goes in.

- **Goal:** Resolve where the TUI shell's own source lives, name that
  repo if it's separate, and confirm the skipper/implementation boundary
  — narrow enough to correct every doc currently asserting the
  shell-lives-in-skipper assumption.
- **In scope:** The shell's repo location; its name if separate; whether
  `skipper` needs a generator command for it; which already-designed
  pieces (catalog client, three dispatch drivers, theme system, built-in
  registry, owner's stash/chat embedded-view adapters) move with it.
- **Out of scope:** Re-opening any of `002`/`007`/`015`/`016`'s settled
  shell *design* decisions (dispatch taxonomy, catalog schema, UX
  convention, notification model) — only their repo location is in
  scope here, not their shape.
- **Success criteria:** A named repo, a confirmed skipper/implementation
  boundary, and a confirmed scaffold-command shape, recorded in this
  doc's Decision Log — then every contradicting doc gets a short
  amendment note, the same way `007` amended `002`.
- **Non-goals:** Designing `kingfish`'s actual internal package layout
  beyond what `002`/`007`/`015`/`016` already specified — this doc is
  about the repo boundary, not new shell design.

## Background & Context

- **Relevant prior work:**
  - `.specify/memory/constitution.md` Principle VIII — "the hybrid TUI
    layer... generated as its own repo per project; not vendored into
    `skipper`" and "individual web applications, separately-deployed
    application binaries, and other projects `skipper` scaffolds or
    builds MUST live and be maintained in their own separate
    repositories — never inside `skipper`." Directly decisive: a
    TUI-shell binary that `skipper` scaffolds is exactly the kind of
    "separately-deployed application binary" this principle already
    covers.
  - [architecture.md](architecture.md) — the layer diagram's "own repo,
    scaffolded by `skipper`, maintained separately... (TUI client repo,
    e.g. pattern from `github.com/monamaret/bbb-le`)" caption, which this
    doc's decision is consistent with; the same doc's later "First major
    release" diagram, which is not (see Problem Statement) and needs
    amending.
  - [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) and
    [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) —
    the shell's confirmed design (interaction models, three dispatch
    mechanisms, five-type catalog schema, compile-time `ViewFactory`
    registry, built-in app registry). Not reopened here.
  - [006-private-chat-application.md](006-private-chat-application.md) §Architecture
    Patterns and [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) —
    both assert (`006` explicitly, `013` implicitly) that the owner's
    chat embedded-view adapter lives in `skipper`'s own repo. Needs
    amending per this doc's decision.
  - [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) —
    asserts the same for stash's owner-side adapter. Needs amending.
  - [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md) —
    the direct structural precedent for this doc's scaffold-command
    question, with one key difference: `biblio-cli` already exists as a
    third-party-from-`skipper`'s-perspective lifecycle tool to wrap.
    `kingfish` does not exist yet — there is no existing CLI to wrap, so
    `skipper`'s generator command must carry the actual starting
    template itself, closer to how `005`'s generic app scaffold provides
    its own CLI+web starting structure than to `017`'s "wrap an existing
    tool" shape.
- **Domain assumption:** `github.com/monamaret/bbb-le` remains a
  **reference pattern** for the shell's interaction-model and deployment
  shape (Wish/Bubble Tea, Cobra, GKE via Kustomize) — it is not the
  literal shell implementation. `kingfish` is a new, separate repo that
  draws on `bbb-le`'s patterns where applicable (e.g. its own deployment
  shape, if `kingfish` itself is ever deployed rather than just
  installed locally) but is not a fork or rename of it.
- **Stakeholders:** mona (sole owner/operator, and the shell's only
  user/operator).

## Libraries & Stack

Not applicable — this doc is a naming/boundary decision, not a
technology comparison. The shell's actual library choices (Bubble Tea,
Lip Gloss, Wish, Wishlist) are already settled in `002`/`007` and are not
revisited here.

## Architecture Patterns

- **The TUI shell is named `kingfish`, at `github.com/monamaret/kingfish`
  (not yet created).** It lives in its own repository, never inside
  `skipper`'s — resolving the Problem Statement's contradiction in favor
  of Principle VIII's explicit text, not the roadmap's drifted wording.
- **What moves into `kingfish`:** everything `002`/`007`/`015`/`016`
  designed as shell-internal — the launcher, the app-catalog client
  (fetch + rook-style local cache with fallback), all three dispatch
  drivers and their shared suspend-and-exec core, the compile-time
  `ViewFactory` registry, the built-in app registry (the Markdown
  reader), the Lip Gloss theme system, the Glow-derived UX convention,
  and the notification/status-bar model (`015`/`016`). Critically, this
  also includes **the owner's embedded-view adapters for stash and
  chat** — `pocket`'s and `rebus`'s client packages get imported into
  `kingfish`, not into `skipper`, since those adapters resolve `viewID`s
  against the shell's own compile-time registry and must be compiled
  into the same binary as the shell itself (`007`'s embedded-view
  mechanism has no dynamic plugin loading). This corrects `006`'s and
  `018`'s Architecture Patterns sections, which currently place those
  adapters in `skipper`'s repo.
- **What stays in `skipper`'s own repo:** nothing shell-side. `skipper`'s
  repo keeps exactly what Principle VIII already scopes it to for every
  other layer: the persisted Kustomize bases for backend apps
  (`deployments/pocket/`, `deployments/rebus/`, `deployments/wishlist/`,
  etc.), the app-catalog *write* path (the admin command that updates
  the GCS/Firestore catalog document), and this research/feature
  pipeline itself. `skipper` never holds shell source, never holds an
  embedded-view adapter, and has no runtime role once `kingfish` is
  installed and running.
- **`skipper` gets a generator command: `skipper tui new <repo-name>`.**
  Unlike `017`'s `skipper site new` (which wraps an already-existing
  `biblio-cli`), there is no pre-existing `kingfish-cli` to wrap —
  `kingfish` doesn't exist yet. `skipper tui new` therefore carries the
  starting template itself: a Go module scaffold with the
  Bubble-Tea/Lip-Gloss/Wish dependency set, the empty `ViewFactory`
  registry, the catalog-client stub (cache-with-fallback, pointed at the
  project's GCS/Firestore catalog location), and the theme-loading
  scaffold — the shell-equivalent of what `005`'s generic app scaffold
  already does for the CLI+web case, not a wrap-and-delegate shape like
  `017`'s. After scaffolding, `skipper` has no further lifecycle role:
  no `skipper tui update`/`deploy` — `kingfish`'s own ongoing development
  (adding the embedded-view adapters for `pocket`/`rebus` as those land,
  building/installing the binary, any future GKE deployment of the
  shell itself if it's ever run remotely rather than installed locally)
  happens entirely inside `kingfish`'s own repo, mirroring `017`'s "no
  `skipper site update`" rationale even though the underlying mechanism
  differs (template-carrying vs. tool-wrapping).
- **One generator invocation, not an ongoing dependency.** Once
  `kingfish new`-equivalent scaffolding has run once, `skipper` and
  `kingfish` have no further coupling beyond the catalog `skipper`
  writes to and `kingfish` reads from (already-decided, unaffected by
  this doc) — consistent with Principle VIII's closing line: "`skipper`
  generates and can update those repos' starting scaffolds, but does not
  vendor or absorb their ongoing source into this repo."

**Tradeoffs:** Moving the embedded-view adapters out of `skipper` and
into `kingfish` means `kingfish`'s repo, not `skipper`'s, is where
`pocket`'s and `rebus`'s public client packages get imported and
exercised — slightly less visibility into shell-side app integration
from `skipper`'s own repo, in exchange for actually matching Principle
VIII instead of carrying a standing exception for the shell alone.
**Integration points:** `kingfish` depends on `pocket`'s and `rebus`'s
published Go client packages (already public, not `internal/`, per
`006`/`018` — that part of those decisions is unaffected) and on the
app-catalog document `skipper`'s admin command writes to (unaffected,
already a cross-repo artifact, not a vendoring relationship).
**Data model implications:** None new — the catalog schema (`007`) and
each backend's data model (`006`, `009`) are unaffected; only which
repo's compile-time registry an embedded view resolves against changes.

## Existing Codebase Evaluation

No `kingfish` code exists yet, and the repo itself hasn't been created.
`bbb-le` remains a reference pattern, not code to fork — same posture
`002`/`architecture.md` already established for the hybrid TUI layer
generally, now made concrete with an actual name and boundary for the
shell specifically.

## Security & Compliance

No new concerns beyond what `002`/`007` already cover (SSH-key-based
auth for SSH-passthrough, WebAuthn for any future web client of an
embedded-view app, no long-lived credentials in the shell). This doc is
purely a repo-boundary decision, not new security design. One
operational note: `kingfish`'s catalog-client read path needs whatever
credential the GCS/Firestore catalog document requires — already
covered by `002`'s existing read-side design, now simply scoped to
`kingfish`'s repo instead of an unspecified location.

## Performance, Reliability & Operations

Not applicable — this doc only concerns where code lives, not how it
runs. No new runtime characteristic is introduced.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Shell-side code (catalog client, dispatch drivers, embedded-view adapters) gets started inside `skipper`'s repo before `kingfish` exists, repeating the exact drift this doc is correcting | Medium | Medium | Don't begin coding any shell-side piece until `github.com/monamaret/kingfish` is actually created and `skipper tui new` has scaffolded it; treat this doc's Decision Log as the authoritative boundary going forward | mona |
| `006`/`013`/`018`/`roadmap.md`/`architecture.md` are left uncorrected, so future work re-reads the stale "lives in skipper's repo" wording and repeats the contradiction | Medium | Medium | Apply the amendment notes listed in this doc's Decision Log to each affected doc now, the same way `007` amended `002` | mona |
| `skipper tui new`'s template drifts out of sync with `002`/`007`/`015`/`016`'s design as those evolve, since there's no `biblio-cli`-style existing tool keeping the template current | Low–Medium | Low–Medium | Accepted for v1 — the template is generated once per shell repo's lifetime (not on every shell change), so drift only matters if a *second* `kingfish`-pattern repo is ever scaffolded; revisit if that happens | mona |

## Open Questions

None — resolved below. One follow-up is **not** a research blocker, just
deferred mechanism detail: whether `skipper tui new`'s template is
generated from an embedded Go template/text-template set within
`skipper`'s own binary, or from a separate template repo `skipper`
clones — an implementation choice for whoever builds the `skipper tui
new` command, not a design decision this doc needs to make.

## Decision Log

- **2026-06-24** — **The TUI shell is named `kingfish`, at
  `github.com/monamaret/kingfish` (not yet created), and lives in its
  own repository — never inside `skipper`'s.** — Rationale: resolves the
  Problem Statement's contradiction in favor of constitution Principle
  VIII's explicit text ("the hybrid TUI layer... generated as its own
  repo... not vendored into `skipper`") and `architecture.md`'s own
  layer diagram, rather than the drifted "living in `skipper`'s own
  repo" wording that crept into `006`/`013`/`018`/`roadmap.md`. — Decided
  by: mona.
- **2026-06-24** — **Everything `002`/`007`/`015`/`016` designed as
  shell-internal moves into `kingfish` — including the owner's
  embedded-view adapters for stash (`pocket`) and chat (`rebus`), which
  were previously (incorrectly) described as living in `skipper`'s own
  repo.** — Rationale: `007`'s embedded-view dispatch mechanism resolves
  `viewID`s against a compile-time registry with no dynamic plugin
  loading — those adapters must be compiled into the same binary as the
  shell, so wherever the shell's source lives is where they live too;
  once the shell moves to `kingfish`, the adapters necessarily move with
  it. — Decided by: mona.
- **2026-06-24** — **`skipper` gets a new generator command, `skipper
  tui new <repo-name>`, which scaffolds `kingfish`'s starting structure
  directly (Go module, Bubble Tea/Lip Gloss/Wish dependencies, empty
  `ViewFactory` registry, catalog-client stub, theme-loading scaffold) —
  not a wrap-an-existing-tool shape like `017`'s `skipper site new`,
  since no `kingfish`-equivalent CLI exists yet to wrap.** `skipper` has
  no lifecycle role after this single invocation — no `skipper tui
  update`/`deploy`; `kingfish`'s own ongoing development happens
  entirely in its own repo. — Rationale: consistent with Principle II
  (single CLI entry point for scaffolding every layer) and Principle
  VIII's "generates... starting scaffolds, but does not vendor or absorb
  ongoing source"; mirrors `017`'s "no ongoing `skipper`-side lifecycle
  command" conclusion even though the underlying mechanism
  (template-carrying, not tool-wrapping) differs because `017`'s
  precedent tool (`biblio-cli`) already existed and `kingfish` does not.
  — Decided by: mona.
- **2026-06-24** — **The following docs need a short amendment note
  pointing to this doc, the same way `007` amended `002`, rather than
  being silently left stale:**
  - [006-private-chat-application.md](006-private-chat-application.md) §Architecture
    Patterns — "the owner's embedded view, living in `skipper`'s own
    repo" → `kingfish`'s own repo.
  - [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) —
    the owner's TUI-embedded chat view reference should clarify it lives
    in `kingfish`, not `skipper`, even though `013`'s own wording never
    explicitly named `skipper` (it only contrasted with `rebus`'s repo).
  - [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) §Architecture
    Patterns — "the owner's TUI view itself... living in `skipper`'s
    repo" → `kingfish`'s own repo.
  - [roadmap.md](roadmap.md) §1 (chat item) — "living in `skipper`'s own
    repo" → `kingfish`'s own repo.
  - [architecture.md](architecture.md) — the "First major release"
    diagram should draw an explicit `kingfish` repo boundary around the
    shell + embedded-view-adapter box, matching the boundary already
    drawn for the backend/web-app/TUI-client layers in the system
    overview diagram above it.
  — Decided by: mona.

## Internal References

- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle
  VIII (repo separation, directly decisive) and Principle II (single CLI
  entry point for scaffolding, basis for `skipper tui new`)
- [architecture.md](architecture.md) — the layer diagram this doc's
  decision is consistent with, and the "First major release" diagram
  that needs amending
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) and [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) —
  the shell design this doc gives a repo boundary to, not reopened
- [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md) —
  the structural precedent for the scaffold-command question, with the
  key difference (no pre-existing tool to wrap) noted above
- [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) and
  [006-private-chat-application.md](006-private-chat-application.md)/[013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) —
  the docs whose owner-side-adapter-location wording this doc corrects
- [roadmap.md](roadmap.md) — the progress tracker entry that needs the
  same correction

## Web Citations

No external web research was needed — this doc resolves an internal
documentation inconsistency using already-cited internal decisions
(`002`, `007`, Principle VIII), not new technical research.

## Appendix

None yet.
