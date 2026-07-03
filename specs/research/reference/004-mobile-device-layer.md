# Research: Mobile device layer — reading from the platform's APIs

**Status:** On hold — premise stale, no live web client exists yet in
this platform to pair a mobile client against (see note below).
**Date:** 2026-06-24
**Owner:** mona

**Staleness note (added 2026-06-24):** This doc's original premise was
that the private chat app ([F-005](../features/005-private-chat-1to1.md))
already had both a TUI and a web client, making mobile the natural third
client. Chat's web client has since been **deferred** — v1 ships TUI-only,
and the dual-thin-client (CLI + web) idea was generalized into
[005-generic-app-scaffold.md](005-generic-app-scaffold.md) instead. That
scaffold's v1 example app also defaults to **static** web mode (no live
backend calls), so there is currently no live, authenticated web client
anywhere in this release to pair a mobile client against. This doc is on
hold until a live web client actually exists (whether chat's eventual one,
or a future live-mode instance of the generic scaffold) — at that point,
the analysis below (WebAuthn extends with no new work; PWA-first
recommended) should still hold, but the "immediate concrete driver"
framing needs re-pointing to whichever app actually has a live web client
first.

## Problem Statement

The platform currently has three layers (constitution Principle VIII):
GKE backend, a web layer (split between standalone content sites on
Cloudflare and backend-application UIs on GCP-native hosting — see
[005-generic-app-scaffold.md](005-generic-app-scaffold.md)), and a hybrid
TUI layer. A fourth surface — a mobile client reading from the platform's
stateless RPC/HTTPS APIs — is not yet covered by any principle or research
doc. This doc was originally triggered by the private chat app
([005-private-chat-1to1.md](../features/005-private-chat-1to1.md)), which
was planned to have both a TUI client and a web client against the same
stateless backend; per the staleness note above, that web client is now
deferred, so this doc's analysis remains a useful default for *whenever*
a live web client exists, but is not actively blocking anything in this
release.

- **Goal:** Determine the lowest-complexity way to add mobile reachability
  for stateless-RPC apps (starting with chat), consistent with Principle I
  (no speculative abstraction) and the WebAuthn-based web auth decision
  just made in [002](002-hybrid-tui-layer.md).
- **In scope:** App-shell options (PWA vs. native vs. cross-platform),
  whether the WebAuthn auth decision extends to mobile, and what's
  genuinely different about mobile vs. web for a stateless RPC client
  (chiefly: push notifications, offline storage, app-store distribution).
- **Out of scope:** A TUI-style SSH-session-served mobile experience —
  none of this platform's SSH-session-served apps (chat is explicitly not
  one, per the supersession in [002](002-hybrid-tui-layer.md); Soft Serve
  and future category-4 apps are terminal-app-shaped, not a natural mobile
  fit) are being redesigned for mobile here.
- **Success criteria:** A confirmed app-shell decision (or an explicit
  "defer until a concrete need" call) recorded in this doc's Decision Log.
- **Non-goals:** Building a generic "mobile SDK" for every future app
  before a second concrete mobile use case exists beyond chat.

## Background & Context

- **Relevant prior work:** [002-hybrid-tui-layer.md → Decision Log](002-hybrid-tui-layer.md#decision-log)
  just established WebAuthn/passkeys, registered into the owner-controlled
  identity registry, as the web client's auth mechanism — this turns out
  to matter directly for mobile (see Architecture Patterns).
  [005-private-chat-1to1.md](../features/005-private-chat-1to1.md) is the
  concrete first consumer: TUI + web today, mobile is the natural next
  surface for the same backend.
- **Domain assumption:** Mobile here means "a way for the owner (and the
  chat app's invited outside-domain participant) to use these stateless
  apps from a phone" — not a new interaction model, just a new client
  shape against APIs that already exist or are already planned.

## Libraries & Stack

Candidate app-shell approaches:

| Approach | Effort | Push notifications | Offline storage | Distribution |
| --- | --- | --- | --- | --- |
| **Progressive Web App (PWA)** — make the existing web client (e.g. chat's React app) installable/responsive; no new codebase | Lowest — reuses the web client almost as-is | Web Push now works on iOS 16.4+ and Android Chrome, but is less consistent/reliable than native push, and historically lagged on iOS | `localStorage`/IndexedDB — sufficient for the same local-cache pattern already used elsewhere (rook-style) | No app store — just a URL; zero review process |
| **React Native** — one cross-platform codebase, TypeScript, shares some logic/patterns with the existing web client | Medium — new codebase, but one, not two; reuses TypeScript skillset (Principle III) | Native push (APNs/FCM) via libraries — reliable | Native local storage (e.g. SQLite, AsyncStorage) | App Store + Play Store — developer accounts, review cycles, signing |
| **Native (Swift/SwiftUI + Kotlin/Compose)** | Highest — two separate codebases/languages, doubles ongoing maintenance for a single owner | Best native push integration, most idiomatic platform UX | Best native storage integration | App Store + Play Store, same distribution overhead as React Native, more idiomatic per-platform polish |

**Selected:** *(pending Decision Log)*
**Rejected (with reason):** Native (Swift + Kotlin) — two codebases in two
languages is a real, ongoing maintenance burden for a single developer
(Principle I), and nothing about this platform's needs so far demands
platform-specific polish that a PWA or React Native couldn't deliver.
**Open questions:** see below.

## Architecture Patterns

- **WebAuthn/passkeys extend to mobile with no new auth mechanism.** Both
  iOS (via `AuthenticationServices`) and Android (via the Credential
  Manager API) have first-class, OS-level passkey support — including
  inside mobile browsers (so a PWA gets this for free) and inside native/
  React Native apps (via passkey-aware libraries, e.g.
  `react-native-passkey`). This means the auth decision just made for web
  in [002](002-hybrid-tui-layer.md) — same identity registry, same
  one-time enrollment-link bootstrap — requires **no mobile-specific auth
  work** regardless of which app-shell approach is chosen.
- **The API itself needs no mobile-specific surface.** Per
  [F-005](../features/005-private-chat-1to1.md), the chat backend is a
  stateless RPC/HTTPS API already designed to serve more than one client
  type (TUI + web). A mobile client is "just another client" against the
  same API, provided the API stays plain HTTP/JSON (or another
  transport-agnostic shape) rather than anything web-specific.
- **The actual differentiator is push notifications, not auth or API
  shape.** This is the one capability a pure web/PWA client structurally
  can't match natively as well as a real app: knowing about a new 1:1
  chat message without the app open. If reliable push turns out to matter
  in practice, that's the concrete trigger to escalate from PWA to React
  Native — not a speculative "mobile users expect an app" assumption.
- **Recommended default: PWA first, explicitly deferring React Native/
  native until push notification reliability (or another concrete gap) is
  actually demonstrated as insufficient.** This mirrors the project's own
  precedent elsewhere (e.g. biblio's storage-escalation philosophy cited
  in this platform's constitution) — start with the simplest viable
  surface, escalate only on a documented, real trigger.
- **Tradeoffs:** A PWA's web-push reliability gap is real but may not
  matter for a single-developer-plus-one-guest 1:1 chat app where checking
  the app periodically is perfectly workable; committing to React Native
  upfront is real ongoing maintenance for a benefit (reliable push) that
  hasn't yet been shown to be necessary.

## Existing Codebase Evaluation

No mobile code exists yet — greenfield. The PWA path reuses
[F-005](../features/005-private-chat-1to1.md)'s planned web client almost
directly (responsive layout + a web app manifest + service worker), so
there's no separate mobile codebase to evaluate yet either way.

## Security & Compliance

- No new auth surface — WebAuthn/passkeys via the same registry as web,
  per [002 → Decision Log](002-hybrid-tui-layer.md#decision-log).
- A PWA inherits the web client's existing security posture (HTTPS,
  httpOnly/secure session cookies) with no additional credential storage
  concerns. A native/React Native app would need its own secure-storage
  review (Keychain/Keystore) for any session material if that path is
  ever taken — not yet relevant under the PWA-first recommendation.

## Performance, Reliability & Operations

- **Push notification reliability** is the one concrete, measurable
  signal that should drive any future escalation past a PWA — not
  vibes-based "users expect a native app."
- **Offline behavior** for a PWA (IndexedDB-backed local cache) follows
  the same rook-style local-cache-with-fallback discipline already
  established elsewhere in this platform — no new pattern needed.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Web push on iOS proves unreliable enough that chat notifications are missed in practice | Medium | Medium | Treat as the concrete trigger to build a React Native client with native push — don't pre-build it speculatively | mona |
| Building React Native (or native) upfront turns into ongoing maintenance burden disproportionate to actual mobile usage | Medium | Medium | PWA-first default; only escalate on a demonstrated gap | mona |
| WebAuthn support gaps on older mobile OS versions/browsers lock out a participant | Low | Medium | Document a minimum supported OS/browser version; this is a real but narrow edge case, not a reason to avoid WebAuthn generally | mona |

## Open Questions

- [ ] **PWA vs. React Native — confirm PWA-first, or skip straight to
  React Native given a known push-notification need for chat?** — owner: mona
- [ ] If PWA-first is confirmed: what's the concrete bar for escalating to
  React Native (e.g. "push notifications missed N times in practice," or
  just "owner's judgment call")? — owner: mona

## Decision Log

*(empty — pending answers to the open questions above)*

> **Format note (2026-07-03):** this doc predated the current
> `research-template.md`; its single `## References` list has been split into
> `## Internal References` + `## Web Citations` to match the template.

## Internal References

- [architecture.md](architecture.md) — platform layers this doc adds a
  fourth surface to
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — WebAuthn web-auth
  decision this doc confirms extends to mobile with no new work
- [F-005 (private chat)](../features/005-private-chat-1to1.md) — the
  concrete first consumer of a mobile client
- [roadmap.md §0](roadmap.md#0-outstanding-design-questions) — tracking
  location for this open item

## Web Citations

The original doc recorded no external web citations. Its mobile-platform-
support claims — iOS 16.4+ Web Push; iOS `AuthenticationServices` and Android
Credential Manager passkey support; `react-native-passkey` — would each need a
cited source before this (on-hold) doc's analysis is acted on.

## Appendix

None yet.
