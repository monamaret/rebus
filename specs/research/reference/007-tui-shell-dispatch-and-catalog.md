# Research: TUI shell — dispatch taxonomy and catalog schema, reconciled against later features

**Status:** Complete — all open questions resolved; ready to update specs/features/001-tui-shell-and-styling.md. Three non-blocking follow-up research items flagged for later (see Open Questions).
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

`001-tui-shell-and-styling.md` (feature) and `002-hybrid-tui-layer.md`
(research) defined the TUI shell with a clean 2-driver model
(SSH-passthrough, stateless-RPC in-process view) and a 3-value catalog
interface schema (`tui-ssh:<port>` / `web:<port>` / none). Three later
docs — `003-off-the-shelf-image-deployment.md`,
`005-generic-app-scaffold.md`, and `006-private-chat-application.md` —
each added requirements that the shell now needs to satisfy, but none of
those requirements were folded back into `001`/`002`. This doc surfaces
the gaps and runs a discovery interview to resolve them before `001` is
updated.

- **Goal:** Reconcile the TUI shell's dispatch model and catalog schema
  against everything decided since `002`, and settle the open design
  questions that surfaced doing so.
- **In scope:** Dispatch mechanism taxonomy (how many distinct ways the
  shell hands off control, and what each needs), catalog interface-type
  schema completeness, multi-screen app support, theme propagation across
  dispatch boundaries, catalog refresh cadence.
- **Out of scope:** Re-deciding the underlying interaction-model split
  itself (categories 1–4, rook/sight vs. bbb-le) — that's settled in
  `002` and not reopened here. This doc is about the shell's *own*
  mechanics for hosting those already-decided patterns, not which pattern
  applies to which app.
- **Success criteria:** Every gap below resolved in this doc's Decision
  Log, then `001-tui-shell-and-styling.md` updated with concrete Scope
  items and acceptance criteria reflecting them.
- **Non-goals:** Designing the exact theme config file format (already
  correctly deferred in `001` as an implementation detail).

## Background & Context — the gaps, traced to their source

- **Gap 0: SSH-passthrough's justification narrowed to one category.**
  SSH-passthrough was originally justified by two apps: chat (category 2)
  and category-4 TUI-only Kubernetes apps (Soft Serve). `006`'s
  supersession moved chat to the stateless rook/sight pattern (in-process
  view) entirely — so across all six v1 features, SSH-passthrough is now
  used by exactly one app: Soft Serve
  ([003-off-the-shelf-image-deployment.md](../features/003-off-the-shelf-image-deployment.md)),
  deliberately chosen as the v1 test case for category 4. **Confirmed
  2026-06-24: keep it in v1** — category 4 (SSH-only-reachable Kubernetes
  apps) was one of the four original TUI usage categories from the
  earliest TUI-layer conversation, independent of chat's design; Soft
  Serve proves a real future need (e.g. a self-hosted git server), not
  leftover scope from chat's superseded design. See Decision Log.
- **Gap 0b: SSH-app networking — one shared Wishlist gateway, not one
  LoadBalancer per app.** Each SSH-reachable app needs an externally
  reachable endpoint; provisioning one GKE `LoadBalancer` Service per app
  is simple but doesn't scale gracefully as more SSH/Wish-pattern apps are
  added (cost and IP-sprawl per app). **Confirmed 2026-06-24: `github.com/charmbracelet/wishlist`
  becomes the shared network-multiplexing layer specifically for
  Wish-pattern SSH apps** (Soft Serve now; future bbb-le-style apps
  later) — one Wishlist instance behind one `LoadBalancer` Service on
  GKE, with Wishlist's own menu/config handling the proxy to whichever
  specific backend. The app catalog still discovers everything
  platform-wide (web, RPC, CLI-exec, *and* SSH apps); SSH-app catalog
  entries simply resolve to the one Wishlist endpoint instead of a
  per-app IP. This does **not** reopen `002`'s earlier Wishlist
  rejection — that rejection was specifically about *building a custom
  discovery gateway* that would duplicate the catalog's job; using
  Wishlist (an existing, proven Charm tool that Soft Serve and other
  Wish-built apps already integrate with natively) to solve the
  *network-multiplexing* problem is a different concern entirely. Non-SSH
  apps (stash, chat, dashboard) are unaffected — they keep the catalog's
  direct RPC/CLI-exec paths. See Decision Log and the amendment in
  [002 → Decision Log](002-hybrid-tui-layer.md#decision-log).
- **Gap 1: a third dispatch mechanism.** `001`'s Scope lists only
  SSH-passthrough and stateless-RPC (in-process Bubble Tea view) drivers.
  `005`'s Decision Log requires shelling out to a local CLI binary for the
  deployment dashboard — neither an SSH handoff nor an in-process view
  switch. `001` doesn't account for this third mechanism at all.
- **Gap 2: incomplete catalog interface-type schema.** `003`'s Decision
  Log defines exactly three interface values: `tui-ssh:<port>`,
  `web:<port>`, or none. Stash (`004`) and chat (`005` feature)'s
  in-process Bubble Tea views, and the dashboard's CLI-exec dispatch
  (`006`'s deployment-admin-dashboard feature), don't map onto any of
  those three values.
- **Gap 3: multi-screen apps unaddressed.** `006-private-chat-application.md`
  confirmed multiple separate 1:1 conversations behind one chat catalog
  entry — implying the chat app owns a small internal screen stack
  (conversation list → conversation), not a single flat view. `001`/`002`
  never specified whether the shell's in-process dispatch supports handing
  control to a nested sub-application or expects one flat view per entry.
- **Gap 4: theme propagation across dispatch boundaries undecided.** `001`
  decided a Lip Gloss-based theme "every plugged-in TUI view reads" — but
  that's only unambiguous for in-process views. For the CLI-exec driver
  (`005`) and the SSH-passthrough driver, the subprocess/remote session is
  a separate process with its own rendering — does the shell communicate
  the active theme to it, or is visual consistency only guaranteed
  in-process?
- **Gap 5: catalog refresh cadence is vague.** `002`'s Decision Log says
  the client "fetch[es] the catalog opportunistically (on launch, or on a
  background refresh)" without picking one. `006` later established a
  manual-check-only, no-background-polling norm for chat sync — worth
  deciding whether the catalog should follow the same norm for
  platform-wide consistency.
- **Gap 6: stale "Confirmed decisions" in 001.** `001` cites only `002` —
  predates `003`, `005`, `006`, each of which added shell-relevant
  requirements not yet reflected there.

## Libraries & Stack

Not applicable — this doc is reconciling already-decided patterns and
the shell's own internal mechanics, not evaluating new external
libraries/services.

## Architecture Patterns

Pending interview answers (Appendix) on the dispatch taxonomy (Q1–Q2),
multi-screen support (Q3), theme propagation (Q4), and catalog refresh
cadence (Q5).

## Existing Codebase Evaluation

No `skipper` TUI shell code exists yet — this doc's findings feed directly
into `001-tui-shell-and-styling.md`'s Scope before any implementation
starts.

## Security & Compliance

No new concerns beyond what `002` already covers (SSH-key-based auth,
no long-lived credentials). The CLI-exec driver (Gap 1) runs a local
binary the owner already trusts (it's `skipper`-deployed), so it doesn't
introduce a new trust boundary the way the SSH-passthrough driver's
remote-host handoff does.

## Performance, Reliability & Operations

Pending interview answer on catalog refresh cadence (Q5) — affects how
"fresh" the launcher's app list is by default.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Implementing `001` against its current, now-stale Scope, then discovering mid-build that a third driver and a richer catalog schema are required | High if not addressed now | Medium–High (rework) | This doc + interview close the gap before `001`'s Scope is touched again | mona |
| Each plugged-in app inventing its own multi-screen navigation convention (back/escape behavior) inconsistently | Medium | Low–Medium (inconsistent UX, not a functional blocker) | Q3 settles whether the shell defines a shared convention | mona |

## Open Questions

All seven interview questions are resolved — see the Decision Log.
Nothing here blocks updating
[specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)
with these concrete specifics. Three follow-up items were flagged during
that update for *future* research (not v1-blocking):

- [x] ~~Generic cross-app notification convention on the status bar~~ —
  **Resolved by [015-shell-notification-and-sync-freshness-model.md](015-shell-notification-and-sync-freshness-model.md):**
  a shared `shell.NotificationMsg` type, delivered via Bubble Tea's
  documented `Program.Send`, that any embedded-view app can emit and the
  shell's root model renders in the status bar — not chat-specific.
- [x] ~~Status bar visibility during SSH-passthrough sessions~~ —
  **Resolved by [015](015-shell-notification-and-sync-freshness-model.md):**
  notifications queue and flush to the status bar the moment the shell
  regains the terminal — no real-time out-of-band signal during the
  session (owner decision: no interruption preferred).
- [x] ~~Non-manual (lazy/configurable) background sync~~ — **Resolved by
  [015](015-shell-notification-and-sync-freshness-model.md):** the
  platform-wide manual-only default is unchanged; the shell instead
  exposes a per-view, opt-in `RefreshOnReentry` config flag (default
  `false`) so an individual app can trigger one sync on view re-entry,
  without changing the global default or introducing a timer.

## Decision Log

- **2026-06-24** — **SSH-passthrough stays in v1**, even though chat's
  supersession (006) leaves Soft Serve as its only v1 justification. —
  Rationale: category 4 (TUI-only Kubernetes apps reachable only via SSH)
  is one of the platform's four original TUI usage categories, independent
  of chat's design history; Soft Serve is a deliberate proof of a real,
  generically-needed future capability, not residual scope from a design
  that moved on. — Decided by: mona.
- **2026-06-24** — **`github.com/charmbracelet/wishlist` is the shared
  network-multiplexing layer for Wish-pattern SSH apps (Soft Serve now,
  future bbb-le-style apps later) — one Wishlist instance behind one GKE
  `LoadBalancer` Service, not one LoadBalancer per app.** The app catalog
  remains the single discovery source for the whole platform; SSH-app
  catalog entries resolve to the one Wishlist endpoint, and Wishlist's own
  menu/config handles the proxy to the specific backend from there. —
  Rationale: avoids per-app LoadBalancer cost/IP-sprawl as more SSH apps
  are added, by reusing an existing, proven tool that Wish-built apps
  (including Soft Serve) already integrate with natively — not by
  building bespoke network-proxy code. This does not reopen `002`'s
  rejection of a custom discovery gateway: that rejection targeted
  *building* a gateway that would duplicate the catalog's discovery job;
  this is *using* an existing tool to solve a distinct problem (SSH
  network multiplexing), with the catalog still the sole discovery
  source. Non-SSH apps (stash, chat, dashboard) are unaffected. — Decided
  by: mona.
- **2026-06-24** — **SSH-passthrough and CLI-exec share one underlying
  "suspend-and-exec external process" driver core, with two thin
  command-builders** (one constructs an `ssh` invocation — now routed
  through Wishlist per the decision above; one constructs a local binary
  invocation) **rather than two separate driver implementations.** —
  Rationale: both mechanisms are mechanically identical (suspend the
  shell's terminal control, exec an external process, wait, restore) —
  only the command being executed differs. Sharing the core avoids
  duplicating suspend/restore/signal-handling logic, and any future fix
  to that core (e.g. terminal-resize forwarding) benefits both dispatch
  types automatically. — Decided by: mona.
- **2026-06-24** — **Catalog interface schema confirmed: five types
  (`ssh-passthrough`, `embedded-view`, `cli-exec`, `web`, `none`), each
  with only its own relevant fields, flat per-type (not deeply nested) —
  matching biblio's flat-config precedent.** `ssh-passthrough` carries
  `host`+`port` (the shared Wishlist gateway — identical across every
  Wish app) plus `target` (the specific Wishlist endpoint name, e.g.
  `soft-serve`) — the `target` field is required because Wishlist's own
  picker menu would otherwise force a redundant second selection after
  the catalog's own list (confirmed via Wishlist's README: a target name
  can be appended as a non-interactive SSH command argument, e.g.
  `ssh -p 2222 wishlist-host soft-serve`, see Web Citations).
  `embedded-view` carries a `viewID` (key into the shell's compile-time
  view registry — no dynamic plugin loading, per Principle I).
  `cli-exec` carries a `binaryPath`. `web` carries a `url`. `none` has no
  fields and isn't written to the catalog at all. — Decided by: mona.
- **2026-06-24** — **Multi-screen apps: a plugged-in embedded-view app can
  own a full nested screen stack (its own Bubble Tea sub-model/state
  machine); the shell shares only a thin global UX convention, not a
  shared screen-stack library, directly adopted from Glow's real,
  proven design** (see Web Citations): `Esc`/`q` steps back one level
  (eventually returning to the launcher when at an app's top level), `?`
  toggles a help-overlay "drawer" listing that screen's available
  keybindings in a two-column layout, and a persistent one-line status
  bar at the bottom of every screen shows context-sensitive
  hints/feedback (e.g. scroll position, or a temporary auto-dismissing
  message after an action). — Rationale: at least two apps (chat's
  conversation-list → conversation, stash's document-list → content)
  already need multi-screen navigation, so a shared *convention* is
  justified now — but the actual stack-management code stays inside each
  app, since building a generic shared library has no demonstrated need
  beyond two cases yet (Principle I). Reusing Glow's exact, already-proven
  convention avoids inventing a new one from scratch. — Decided by: mona.
- **2026-06-24** — **Theme propagation splits by dispatch mechanism, not a
  single yes/no: CLI-exec gets theme consistency for free (the shell and
  the CLI binary — always a `skipper`-built process — read the same local
  theme config file via shared theme-loading code, no explicit
  propagation needed); SSH-passthrough theme propagation does not apply
  for v1 (off-the-shelf apps like Soft Serve render entirely server-side
  with their own baked-in styling, per `003`'s "never modified, only
  configured" decision — there's no theme to propagate to).** —
  Rationale: CLI-exec only ever targets `skipper`'s own binaries, so
  sharing config is trivial; SSH-passthrough's v1 target is a third-party
  image skipper doesn't control or render. A future `skipper`-built
  Wish-pattern app could in principle receive a theme over the SSH
  session, but that's deliberately deferred — no such app exists yet. —
  Decided by: mona.
- **2026-06-24** — **Catalog refresh is launch-only, plus a manual refresh
  command/keybinding — no automatic background refresh while the launcher
  sits idle.** — Rationale: matches chat's manual-check-only sync
  precedent (`006`) and the platform's broader no-ambient-process bias; a
  background timer is unjustified complexity for a single-user tool where
  the user can trivially force a re-check (or just restart the shell)
  right after deploying something new. — Decided by: mona.
- **2026-06-24** — **`001`'s acceptance criteria are expanded to cover all
  three dispatch mechanisms (SSH-passthrough via Wishlist, in-process
  view, CLI-exec) and the headless/`none` case explicitly never appearing
  in the launcher's list.** — Rationale: the original two-driver
  acceptance criteria predate this doc's findings; leaving them as-is
  would under-test the now-confirmed three-mechanism, five-type-schema
  reality. — Decided by: mona.
- **2026-06-24** — **`002`'s "clean 2-way split" Decision Log entry is
  amended (not reversed) to clarify it describes two *interaction-model*
  categories (stateless-client vs. SSH-session-served), which now map to
  *three* dispatch *mechanisms* at the shell-implementation level
  (in-process view and CLI-exec both serve the stateless-client category;
  SSH-passthrough serves the SSH-session-served category).** — Rationale:
  the original wording could be misread as a 1:1 category-to-mechanism
  mapping, which `005`'s CLI-exec driver already broke; clarifying this in
  `002` prevents future confusion rather than leaving the apparent
  mismatch unexplained. — Decided by: mona.
- **2026-06-24 (gap found during cross-doc consistency review)** —
  **Purely local apps with no remote component at all (the Markdown
  reader) are discoverable via a small, fixed "built-in app registry"
  compiled into the shell binary itself — shown in the launcher's list
  alongside the dynamic, catalog-driven entries — not via a catalog
  entry.** Built-in apps dispatch through the same in-process
  embedded-view mechanism as catalog-driven ones (same `ViewFactory`
  registry, same Glow-derived UX convention), the only difference is
  *where the launcher learns the entry exists*: a small hardcoded list
  vs. the fetched/cached catalog. — Rationale: the catalog's `embedded-
  view`/`cli-exec`/`ssh-passthrough`/`web` schema describes *independently
  deployed* apps reachable over a network or subprocess boundary — the
  Markdown reader is neither; it ships as part of the shell binary itself
  and has nothing to deploy or discover remotely. Forcing it through the
  catalog schema (e.g. inventing a sixth "local" interface type) would
  conflate "what's independently deployable" with "what's always
  available," which are genuinely different concerns. A short, compiled-
  in list of built-ins is simpler and avoids that conflation (Principle
  I). — Decided by: mona.
- **2026-06-24 (resolves the presentation half of the built-in-registry
  decision above)** — **Built-ins and catalog entries render as one
  flat, unified launcher list — not sectioned — with built-ins listed
  first, ahead of catalog entries.** A catalog entry whose `viewID`
  collides with a reserved built-in `viewID` is a configuration error:
  the shell rejects/logs it rather than silently resolving to one or the
  other. Search/filter across the combined list is explicitly deferred —
  not built for v1. — Rationale: with one built-in (the Markdown reader)
  and a handful of catalog apps in this release, sectioning or search
  would be visual/feature overhead with no real benefit yet; revisit only
  once the combined list is actually long enough to need either
  (Principle I). Built-in `viewID`s are effectively reserved
  identifiers in the shell's compile-time registry — treating a
  collision as an error (not a silent override) avoids ambiguous
  dispatch behavior. — Decided by: mona.

## Internal References

- [001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md) — the feature this research will update
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — underlying interaction-model and catalog-architecture decisions, not reopened here
- [003-off-the-shelf-image-deployment.md](../features/003-off-the-shelf-image-deployment.md) — source of the catalog interface-schema gap (Gap 2)
- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — source of the CLI-exec driver gap (Gap 1) and the MCP terminal-visualization tier
- [006-private-chat-application.md](006-private-chat-application.md) — source of the multi-screen-app gap (Gap 3) and the manual-sync-cadence precedent (Gap 5)
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias), relevant to how much shared-convention machinery is worth building now vs. deferring

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `charmbracelet/wishlist` README | https://github.com/charmbracelet/wishlist | 2026-06-24 | Confirms Wishlist supports direct, non-interactive addressing of a named endpoint via an SSH command argument (e.g. `ssh -p 2222 wishlist-host soft-serve`), which is why the catalog's `ssh-passthrough` interface entries need a `target` field — without it, selecting an app from skipper's catalog would still hit Wishlist's own interactive picker a second time. |
| `charmbracelet/glow` — Stash View (DeepWiki) | https://deepwiki.com/charmbracelet/glow/2.2.1-stash-view | 2026-06-24 | Confirms Glow's two-screen model (filterable file list → document view) as a real precedent for chat's/stash's list-then-detail navigation need. |
| `charmbracelet/glow` — Document View (DeepWiki) | https://deepwiki.com/charmbracelet/glow/2.2.2-document-view | 2026-06-24 | Source of the adopted global convention: `Esc`/`q` returns to the prior screen, `?` toggles a two-column help overlay ("the drawer"), and a persistent one-line status bar shows context/feedback — quoted directly in this doc's Decision Log. |

## Appendix

### Discovery Interview

**A. Dispatch taxonomy** — ✅ fully resolved, see Decision Log

1. ~~Confirm three mechanisms, not two, keep SSH-passthrough, and clarify
   002's framing?~~ — **Confirmed: three mechanisms (SSH-passthrough,
   in-process view, CLI-exec); SSH-passthrough stays in v1; `002` amended
   to clarify two interaction-model categories now map to three
   shell-level dispatch mechanisms, not a 1:1 category-to-mechanism
   relationship.**
2. ~~Same underlying driver, or two separate implementations?~~ —
   **Resolved: one shared "suspend-and-exec" driver core**, with two thin
   command-builders (SSH invocation vs. local binary invocation). See
   Decision Log.

**B. Catalog schema** — ✅ resolved, see Decision Log

3. ~~What interface-type values does the catalog actually need?~~ —
   **Five types, flat per-type fields: `ssh-passthrough` (host, port,
   target), `embedded-view` (viewID), `cli-exec` (binaryPath), `web`
   (url), `none` (no fields, not written to the catalog).**

**C. Multi-screen apps** — ✅ resolved, see Decision Log

4. ~~Flat view or nested screen stack, and shared convention or not?~~ —
   **Apps own a full nested screen stack; the shell shares only a thin
   global UX convention adopted from Glow's real design: `Esc`/`q` back,
   `?` help-overlay drawer, persistent status bar — no shared
   stack-management library.**

**D. Theme propagation** — ✅ resolved, see Decision Log

5. ~~Propagate theme to subprocesses?~~ — **Splits by mechanism: CLI-exec
   gets it for free (shared config file, both processes are
   `skipper`-built); SSH-passthrough doesn't apply for v1 (off-the-shelf,
   server-rendered, not modified).**

**E. Catalog refresh cadence** — ✅ resolved, see Decision Log

6. ~~Launch-only or background refresh?~~ — **Launch-only, plus a manual
   refresh command/keybinding. No background timer.**

**F. Acceptance-criteria coverage** — ✅ resolved, see Decision Log

7. ~~Expand acceptance criteria?~~ — **Yes — all three drivers plus the
   headless/`none` case.**
