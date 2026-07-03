# Research: Notification and local-app-screen customization — emoji/icon theming

**Status:** Complete — schema and resolution order resolved directly from the "light theming" framing and existing `007`/`015` mechanisms; ready to update specs/features/001-tui-shell-and-styling.md and specs/features/005-private-chat-1to1.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[015-shell-notification-and-sync-freshness-model.md](015-shell-notification-and-sync-freshness-model.md)
defined `shell.NotificationMsg{Source, Text, Level, Timestamp}` as the
generic cross-app notification mechanism, and
[007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md)
already established a configurable, Lip-Gloss-based theme file every
view reads — but neither addresses *visual* customization of
notifications or other small per-app/per-element details beyond color
styling. This doc designs a light theming extension specifically for:
configurable emoji/icons per app and per notification type, and other
small configurable elements (the concrete example given: rendering a
chat notification with an emoji standing in for the first character of
the sender's username, rather than a fixed icon).

- **Goal:** A concrete, lightweight config schema (extending the
  existing theme file, not a new config system) for per-app/per-type
  notification icons and username-initial-emoji substitution, narrow
  enough to update `001-tui-shell-and-styling.md` and
  `005-private-chat-1to1.md` with real specifics.
- **In scope:** Notification icon/emoji configuration (by app `Source`
  and by notification `Type`); the username-initial-emoji mechanism;
  where this config lives and how it composes with the existing theme
  file; correctness concerns specific to emoji in a terminal renderer.
- **Out of scope:** General-purpose theming beyond notifications/icons
  (color palettes, borders, etc. — already covered by `007`'s existing
  Lip-Gloss theme system, not reconsidered here). A full templating
  engine for arbitrary notification text composition — considered and
  rejected below in favor of a lighter, explicit-fields schema.
- **Success criteria:** A confirmed config schema and rendering mechanism,
  recorded in this doc's Decision Log.
- **Non-goals:** Building a generic "every app can define arbitrary
  custom UI chrome" system — this doc is scoped to notification icons and
  the specific username-initial pattern, not a general extensibility
  framework.

## Background & Context

- **Relevant prior work:**
  - [015 → Architecture Patterns](015-shell-notification-and-sync-freshness-model.md#architecture-patterns) — `shell.NotificationMsg`, the message type this doc's icon config attaches to.
  - [007 → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log) — the existing Lip-Gloss-based theme config file every view (and now the status bar) already reads; this doc extends that file rather than introducing a second config surface.
  - `.specify/memory/constitution.md Principle I` — minimal-dependency, single-owner bias — directly shapes the "explicit fields, not a template engine" choice below.
- **Domain assumption:** This is a single-owner, personal config file the
  owner edits by hand occasionally — not a config surface other people
  need to learn or that needs runtime validation tooling beyond basic
  parse-error reporting.
- **Stakeholders:** mona (sole shell user and sole editor of this config).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Explicit config fields (`icon`, `usernameInitialEmoji` map) — no templating | Compose the rendered notification line from a fixed, small set of configurable slots | N/A (in-house schema) | N/A | N/A | High | Matches "a *light* theming system" as framed — a config file with a few clear fields, not a syntax to learn. Selected. |
| Go `text/template` per app/type, referencing `.Text`/`.Source`/`.Fields` | Fully general notification-line composition | BSD-3-Clause (stdlib) | Mature | Go team | Rejected | More powerful (arbitrary composition logic) but heavier for a single-owner config file — requires learning Go template syntax for a personal preference setting; explicit fields cover the stated use cases without it. Revisit only if a real need for more composition flexibility than fixed fields can express actually shows up. |
| `mattn/go-runewidth` / `rivo/uniseg` (terminal display-width calculation for emoji/wide characters) | Correct status-bar alignment when arbitrary emoji are inserted | MIT | Mature | Actively maintained | Not adopted as a new dependency — see Risks | `lipgloss` (already a dependency per `007`) has its own width-calculation path (`charmbracelet/x/ansi`), but it has **documented, open bugs specifically around emoji width** (see Web Citations) — a real, known limitation, not a hypothetical. Adding a second width library wouldn't fix `lipgloss`'s own rendering, since `lipgloss.Width`/styling is what actually lays out the status bar — so this doc does not introduce a new width-calculation dependency; it flags the existing risk instead (see Risks & Mitigations). |

**Selected:** Explicit config fields (no templating engine) for icon/
emoji composition, extending the existing theme file from `007`.
**Rejected (with reason):** Go `text/template` (more power than needed
for a personal config); a second width-calculation library (wouldn't
actually fix `lipgloss`'s own emoji-width handling, which is the actual
rendering path). See table.
**Open questions:** none for this section — resolved directly from the
"light/explicit" framing and the documented `lipgloss` width-limitation
constraint, not an owner preference call.

## Architecture Patterns

- **Extend `shell.NotificationMsg` with two optional fields: `Type
  string` and `Fields map[string]string`.** `Type` is a free-form,
  app-defined subtype string (distinct from the existing `Level`
  severity field) — e.g. chat might emit `Type: "new-message"` or
  `Type: "invite-accepted"` — giving icon config something more granular
  than severity to key off. `Fields` carries small, app-supplied
  structured data a notification's rendering might reference — for chat,
  `Fields["username"]` is the concrete example driving this doc.
- **Theme config gains a `notifications` section, cascading from most-
  to least-specific:**
  ```yaml
  notifications:
    chat:                       # keyed by Source
      new-message:              # keyed by Type
        useUsernameInitial: true  # render Fields["username"]'s first
                                   # character via the map below, instead
                                   # of a static icon
      default:
        icon: "💬"               # fallback icon for any other chat Type
    stash:
      default:
        icon: "📝"
    default:
      icon: "🔔"                 # global fallback if Source has no entry
  usernameInitialEmoji:
    m: "🌙"
    a: "🍎"
    # ... owner-configured, case-insensitive lookup on the first
    # character of Fields["username"]; an unmapped character falls back
    # to the notification's own resolved icon (chat's `default.icon`,
    # then the global `default.icon`), never an error
  ```
  Resolution order for a given notification: (1) if its app+type entry
  sets `useUsernameInitial: true` and `Fields["username"]` is present,
  look up its lowercased first character in `usernameInitialEmoji`; (2)
  otherwise, or if that lookup misses, use the app+type's `icon`; (3) if
  no app+type entry exists, use the app's `default.icon`; (4) if no app
  entry exists at all, use the global `default.icon`.
- **This is additive to `001`'s existing theme file, not a new config
  file or system** — the same single config the launcher and every
  embedded view already read gains one more top-level section, read by
  the status bar's `shell.NotificationMsg` renderer specifically.

**Tradeoffs:** A fixed, explicit schema (icon + username-initial-map)
covers the concrete use cases named without the flexibility a template
engine would offer for cases not yet imagined — an accepted limitation
consistent with "light," revisited only if a real need for more
composition power shows up.
**Integration points:** Resolution happens in the same status-bar
rendering code `015` already introduces, reading the same theme-config
load path `007` already established — no new config-loading mechanism.
**Data model implications:** None server-side — this is entirely local,
in-process rendering config; `Fields` on `shell.NotificationMsg` is an
in-memory Go struct field, never persisted or sent over the wire.

## Existing Codebase Evaluation

No `skipper` shell/theme code exists yet. `007`'s already-decided
Lip-Gloss theme file is the direct extension point; no reference repo
(biblio, bbb-le, rook-reference, sight) has an equivalent per-notification
icon-theming feature to draw a concrete precedent from — this is a novel
addition specific to this platform's own notification mechanism (`015`),
not adapted from an external pattern.

## Security & Compliance

- Purely local, cosmetic config — no new credential, network call, or
  data exposure. `Fields` values (e.g. a username) are already visible to
  the owner via the notification's own `Text`; this doc doesn't expose
  anything not already shown.

## Performance, Reliability & Operations

- Resolution is a handful of map lookups per notification render — no
  measurable performance concern at this platform's scale (single user,
  occasional notifications).
- A malformed/missing theme-config `notifications` section degrades to
  the hardcoded global default (🔔) rather than failing to render or
  erroring — consistent with the cache-with-fallback discipline already
  used elsewhere on this platform (catalog, stash).

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| `lipgloss`'s documented emoji-width-calculation bugs (see Web Citations) cause status-bar misalignment when an owner-configured emoji has an unexpected display width | Medium | Low–Cosmetic | Accepted as a known, upstream limitation — not something this doc's config schema can fix; if a specific configured emoji visibly misaligns the status bar in practice, swap it for a different one (a config change, not a code fix) | mona |
| An owner-configured `usernameInitialEmoji` map has a gap for a real participant's username initial, silently falling back to the generic icon | Low | Low | Working as designed — silent, graceful fallback is the explicit behavior (see Architecture Patterns resolution order), not a bug | mona |

## Open Questions

None — the schema, resolution order, and field additions are all
resolved directly above from the "light theming" framing and the
existing `015`/`007` mechanisms this extends; no genuine owner-preference
call remains unresolved.

## Decision Log

- **2026-06-24** — **Notification icon theming is config-only (icon
  string + a `usernameInitialEmoji` character map), not a templating
  engine** — extending `007`'s existing theme file with a new
  `notifications` section, not a new config system. — Rationale: matches
  the "light theming system" framing; a template engine is more power
  than a personal config file needs for the stated use cases. — Decided
  by: mona.
- **2026-06-24** — **`shell.NotificationMsg` (from `015`) gains two
  optional fields: `Type string` (app-defined subtype, separate from the
  existing `Level` severity) and `Fields map[string]string` (small
  app-supplied structured data, e.g. a sender's username).** —
  Rationale: gives icon config something more granular than severity to
  key off, and a place for the username-initial example's input data,
  without inventing a new message type. — Decided by: mona.
- **2026-06-24** — **Resolution order: app+type → app default → global
  default, with `usernameInitialEmoji` checked first when an app+type
  entry opts into it via `useUsernameInitial: true`.** A missing/
  unconfigured entry at any level always falls back gracefully, never
  errors. — Decided by: mona.

## Internal References

- [015-shell-notification-and-sync-freshness-model.md](015-shell-notification-and-sync-freshness-model.md) — `shell.NotificationMsg`, the message type this doc extends
- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) — the existing Lip-Gloss theme config file this doc's `notifications` section is added to
- [specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md) — the feature item this research will refine with the theme-config extension
- [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md) — the feature item that will populate `Type`/`Fields` (e.g. `username`) on its emitted notifications
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias), directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `charmbracelet/lipgloss` — Issue #562, "Emoji/Unicode Width Calculation Causes Layout Misalignment" | https://github.com/charmbracelet/lipgloss/issues/562 | 2026-06-24 | Confirms a real, currently-open bug in `lipgloss`'s emoji width calculation (via `charmbracelet/x/ansi`) — the basis for this doc's Risk entry and for not introducing a second width-calculation library (it wouldn't fix the actual rendering path). |
| `charmbracelet/lipgloss` — Issue #55, "Width issue with certain emoji" | https://github.com/charmbracelet/lipgloss/issues/55 | 2026-06-24 | A second, independent confirmation of the same class of emoji-width bug in `lipgloss`, reinforcing that this is a known, recurring upstream limitation rather than an edge case. |

## Appendix

None yet.
