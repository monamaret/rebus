# Research: Shell notification convention, SSH-passthrough visibility, and sync freshness

**Status:** Complete — all three follow-ups from `007` resolved (two directly from Bubble Tea's documented API, two via owner decision); ready to update specs/features/001-tui-shell-and-styling.md and specs/features/005-private-chat-1to1.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[007-tui-shell-dispatch-and-catalog.md → Open Questions](007-tui-shell-dispatch-and-catalog.md#open-questions)
flagged three related, still-unresolved follow-ups about the shell's
status bar and refresh model: (1) a generic cross-app notification
convention so chat (and any future app) isn't a special case, (2) the
fact that SSH-passthrough sessions hand the entire terminal to an
external process, leaving the shell unable to render *anything*
(including a notification) until that session ends, and (3) whether a
lazy/configurable refresh would be worth adding beyond the platform-wide
manual-only default. All three concern the same underlying thing — how
the shell handles activity that happens while it isn't actively rendering
or being looked at — so this doc resolves them together.

- **Goal:** A concrete notification message convention, a resolution for
  the SSH-passthrough visibility gap, and a decision on lazy refresh —
  narrow enough to update `001-tui-shell-and-styling.md` and
  `005-private-chat-1to1.md` with real specifics.
- **In scope:** The shared notification message type and how apps emit
  it; what happens to a notification while the shell isn't rendering
  (SSH-passthrough/CLI-exec); whether to add a lazy, event-triggered
  refresh alongside the existing manual-only default.
- **Out of scope:** Chat's own in-app notification UI (already separately
  deferred per `006`'s Open Questions) — this doc is the *shell-level*
  convention those notifications would surface through, not chat's
  internal UX. Re-opening the manual-only default itself as the
  *primary* behavior — this doc considers only an *additional*,
  event-triggered lazy refresh layered on top, not replacing it with
  background polling.
- **Success criteria:** All three follow-ups resolved (directly, or via
  owner decision) and recorded in this doc's Decision Log.
- **Non-goals:** Building the notification system before a second
  notification-emitting app beyond chat exists to validate the generic
  shape — the convention is designed now (since chat needs it), but no
  speculative extra apps are invented to test it further.

## Background & Context

- **Relevant prior work:**
  - [007 → Open Questions](007-tui-shell-dispatch-and-catalog.md#open-questions) — the three follow-ups this doc resolves, verbatim.
  - [007 → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log) — the suspend-and-exec driver core (shared by SSH-passthrough and CLI-exec) that hands the terminal to an external process, the root cause of follow-up 2.
  - [006-private-chat-application.md → Open Questions](006-private-chat-application.md#open-questions) — chat's own in-app notification UI, separately deferred; this doc is the shell-level layer underneath that.
  - `.specify/memory/constitution.md Principle I/II` — minimal-dependency bias and "orchestrate, don't reimplement" — both bear on the out-of-band-signal mechanism chosen below.
- **Domain assumption:** `skipper`'s TUI shell runs on the owner's own
  development machine (macOS or Linux, per the platform's stated scope),
  not a constrained/headless-only environment — this makes OS-level
  desktop notifications a viable option to consider, unlike on a pure
  remote/headless server.
- **Stakeholders:** mona (sole shell user).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `(*tea.Program).Send(msg tea.Msg)` | Inject a notification message into the shell's Update loop from a background goroutine | MIT (bubbletea) | Mature, documented core API | Actively maintained by Charm | High | Confirmed via Bubble Tea's own package docs (see Web Citations): "allowing messages to be injected from outside the program... safe to send messages after the program has exited." Exactly the mechanism needed for a background notification watcher to reach the shell's Update loop regardless of what's currently rendering. |
| Shell out to OS-native notifiers (`notify-send` on Linux, `osascript -e 'display notification'` on macOS) | Out-of-band signal while the shell isn't rendering (SSH-passthrough/CLI-exec) | N/A (orchestrates existing OS tools) | Mature, standard OS tooling | OS-maintained | Rejected (by owner decision) | Technically the right shape (no new Go dependency, consistent with Principle II) — but the owner chose queue-and-flush-on-return only, no real-time interrupt during a session (see Decision Log). Documented here in case the preference changes later. |
| Third-party Go desktop-notification library (e.g. a `beeep`-style package) | Same purpose, via a library instead of shelling out | Varies | Varies | Varies | Rejected | Moot once the OS-notification approach itself was declined; would have added a dependency for no benefit over shelling out anyway. |
| Raw terminal bell (`\a` written directly to the tty) | Same purpose, via the terminal itself | N/A | N/A | N/A | Rejected | Requires writing to the controlling terminal's file descriptor while an exec'd foreground process (SSH session, CLI tool) also owns it — a real interleaving/corruption risk with no clean way to guarantee it doesn't garble the foreground process's own output. |

**Selected:** `tea.Program.Send` for in-process notification delivery to
the shell's Update loop, with arriving notifications queued and flushed
to the status bar the moment the shell regains the terminal — no
real-time out-of-band signal during SSH-passthrough/CLI-exec (owner
decision, see Decision Log).
**Rejected (with reason):** OS-native notifiers, third-party Go
notification libraries, raw terminal bell. See table.
**Open questions:** none for this section — the mechanism question was
resolved directly from Bubble Tea's documented API; the "should it
interrupt in real time" question was a genuine owner preference, now
resolved (Decision Log).

## Architecture Patterns

### Generic cross-app notification convention (resolves follow-up 1)

- **A shared `tea.Msg` type, `shell.NotificationMsg{Source, Text, Level,
  Timestamp}`, defined in the shell's own package and imported by any
  embedded-view app that wants to notify.** An embedded-view app (e.g.
  chat's owner-side view) emits this the same way any Bubble Tea
  component emits a message — returned from a `tea.Cmd`, bubbled up
  through the normal `Update` chain to the shell's root model, which
  appends it to a small in-memory queue and renders the most recent
  entry in the status bar. This directly matches Bubble Tea's own
  message-passing idiom — no new framework concept, just one new message
  type apps opt into.
- **Out-of-process apps (SSH-passthrough, CLI-exec) cannot emit this
  message directly** — they aren't Bubble Tea components and aren't
  running inside the shell's Update loop while suspended. For these, the
  notification source is necessarily the *backend* the app talks to
  (e.g. `rebus` for chat), not the foregrounded process itself. A
  background goroutine in the shell — started once at launch, **not
  suspended by `tea.ExecProcess`/the suspend-and-exec driver**, since only
  the foreground rendering loop is suspended, not the whole `skipper`
  process (confirmed via Bubble Tea's own `ExecProcess` behavior, see Web
  Citations) — polls for new notifications (manually triggered, per the
  platform's existing manual-sync default — see follow-up 3 below for
  whether this poll itself should ever be non-manual) and calls
  `program.Send(shell.NotificationMsg{...})` when one arrives. This is
  exactly `tea.Program.Send`'s documented "interoperability" use case.

### SSH-passthrough visibility (resolves follow-up 2)

- **Queue-and-flush-on-return only — no real-time out-of-band signal
  during the session (owner decision).** `shell.NotificationMsg`s that
  arrive while the shell isn't actively rendering (suspended for
  SSH-passthrough/CLI-exec) are queued by the root model regardless —
  Bubble Tea's `Program.Send` is documented to work for messages sent at
  any time, queued until the Program is actively processing — and
  immediately rendered in the status bar the moment control returns. The
  OS-level-desktop-notification option (a real-time interrupt mid-session)
  was considered and explicitly declined: the owner prefers no
  interruption while inside an SSH-passthrough or CLI-exec session,
  discovering anything new only once back at the shell.
- This resolves the original conflict directly: "accept that
  SSH-passthrough sessions are outside the notification system's reach"
  is no longer necessary — nothing is lost, it simply isn't surfaced
  until the shell can render again, which is the explicitly preferred
  behavior, not a fallback being settled for.

### Sync freshness (resolves follow-up 3)

- **The platform-wide manual-only default is unchanged — but the shell
  exposes a per-view config option, off by default, that lets an
  individual embedded view opt into refresh-on-reentry for itself.**
  Resolved per the owner's explicit answer: keep manual as the global
  default, but "give option for apps to configure a refresh-on-view-
  reentry" — i.e. this is each app's own choice, not a platform-wide
  behavior change. Concretely: the embedded-view registration
  (`ViewFactory`, per `001`/`007`) gains an optional
  `RefreshOnReentry bool` field; when `true`, the view's focus-regain
  hook triggers exactly one sync call (the same call the manual refresh
  keybinding already triggers — not a new sync mechanism, just one more
  trigger point) whenever the owner navigates back into that view. When
  `false` (the default for any view that doesn't set it), behavior is
  unchanged from the existing manual-only default. No timer, no
  background loop while a view is merely sitting open and unfocused —
  the trigger is always a real navigation event, never time-based.

**Tradeoffs:** Declining the OS-notification path means a notification
arriving early in a long SSH-passthrough session won't be seen until the
session ends, however long that takes — an accepted, deliberate tradeoff
(no interruption) rather than an oversight. The per-view
`RefreshOnReentry` flag adds one small, opt-in config field per
embedded-view registration — negligible cost for the flexibility of
letting each app decide its own freshness behavior without changing the
platform-wide default.
**Integration points:** `shell.NotificationMsg` and the queue live in the
shell's root model — the same place the status bar's existing
passive-display state already lives, extended rather than replaced.
`RefreshOnReentry` lives on the same `ViewFactory`/catalog-entry
registration struct already defined in `001`/`007`, not a new
registration mechanism.
**Data model implications:** none — this is purely an in-process,
in-memory shell concern; no new backend state.

## Existing Codebase Evaluation

No `skipper` shell code exists yet. Bubble Tea's own `Program.Send` and
`ExecProcess` APIs are the direct, literal mechanisms this doc's design
is built on — not a `skipper`-internal pattern, an external framework's
own documented idiom adopted as the project's convention.

## Security & Compliance

- Shelling out to OS-native notification tools passes only the
  notification's own text (already locally-known, non-sensitive at the
  shell layer) — no new credential or network exposure.
- No change to any existing auth/identity posture — this doc is purely
  about local UI/UX plumbing.

## Performance, Reliability & Operations

- The background notification-watcher goroutine's own poll is itself
  manual-triggered (consistent with the platform-wide default) unless
  the owner opts into the lazy-refresh-on-view-reentry behavior above —
  no new always-on background timer either way.
- `tea.Program.Send` is documented as safe to call even after the
  Program has exited (a no-op) — no special shutdown-ordering logic
  needed for the watcher goroutine.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| The notification queue grows unbounded if the owner stays inside a very long SSH-passthrough session with many notifications arriving | Low | Low | Cap the queue (e.g. keep only the most recent N, or collapse to "N new notifications" once a small threshold is hit) — an implementation detail, not a design blocker | mona |
| A view's `RefreshOnReentry: true` masks a slow sync call behind what looks like instant navigation, if the sync isn't fast | Low | Low–Medium | The sync call is the same one already used for manual refresh, already expected to be fast enough for an interactive keybinding — no new performance requirement introduced | mona |

## Open Questions

None — both questions originally posed here are resolved directly below.

## Decision Log

- **2026-06-24** — **No real-time out-of-band signal (e.g. an OS-level
  desktop notification) during SSH-passthrough/CLI-exec sessions —
  notifications queue and flush to the status bar only once the shell
  regains the terminal.** — Rationale: the owner prefers no interruption
  while inside a suspended session; discovering new activity on return is
  the preferred experience, not a fallback being settled for. — Decided
  by: mona.
- **2026-06-24** — **The platform-wide manual-only sync default is
  unchanged; the shell instead exposes a per-view, opt-in
  `RefreshOnReentry` config flag (default `false`) that lets an
  individual embedded view trigger one sync call when the owner
  navigates back into it.** — Rationale: keeps the platform-wide default
  exactly as already decided, while letting each app decide its own
  freshness tradeoff for itself, per the owner's explicit answer ("manual
  default but give option for apps to configure a refresh-on-view-
  reentry"). — Decided by: mona.

## Internal References

- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) — the source of all three follow-ups this doc resolves, and the suspend-and-exec driver this doc's SSH-passthrough resolution works around
- [006-private-chat-application.md](006-private-chat-application.md) — chat's own deferred in-app notification UI, the first real consumer of this doc's shell-level convention
- [specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md) — the feature item this research will refine with the notification convention and (if confirmed) the lazy-refresh behavior
- [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md) — the feature item that will emit `shell.NotificationMsg` for new messages
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and Principle II (orchestrate, don't reimplement), both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `github.com/charmbracelet/bubbletea` package docs — `Program.Send` | https://pkg.go.dev/github.com/charmbracelet/bubbletea | 2026-06-24 | Confirms `Program.Send(msg Msg)` as the documented mechanism for injecting a message into a running program from an external goroutine, and that it's safe to call even after the program exits — the basis for this doc's background-watcher-to-shell delivery design. |
| Web search — Bubble Tea `ExecProcess`/`Suspend` and background goroutines | (search query, no single URL) | 2026-06-24 | Confirms `ExecProcess` only suspends the foreground rendering loop, not the whole process — background goroutines (e.g. a notification watcher) continue running and can communicate back via the message channel, which is what allows a watcher to survive an SSH-passthrough/CLI-exec session and still reach the shell. |

## Appendix

None yet.
