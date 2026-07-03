# Research: Guest standalone chat client (`bateau`) — feature scope, auth mechanism, and session-state UX

**Status:** Complete — all three flagged follow-ups resolved directly from established Go SSH-tooling patterns and existing platform precedents; ready to update specs/features/005-private-chat-1to1.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[006-private-chat-application.md → Open Questions](006-private-chat-application.md#open-questions)
flagged three related, still-unresolved follow-ups about the invited
outside-domain participant's standalone client — since named **`bateau`**,
to live at `github.com/monamaret/bateau` (not yet created; see
[006 → Decision Log](006-private-chat-application.md#decision-log)): (1)
the UX when the SSH signing key/agent isn't available at auth time, (2)
whether SSH-key auth is still the right fit specifically for this client,
and (3) the client's full feature scope (commands/screens/distribution),
which is currently just one Scope bullet in
[005-private-chat-1to1.md](../features/005-private-chat-1to1.md). All
three are about the same client, so this doc resolves them together.

- **Goal:** A concrete design for `bateau` — its full command/screen
  surface, distribution story, and auth/session-state UX — narrow enough
  to update `005`'s feature spec with real specifics.
- **In scope:** `bateau`'s commands/screens, distribution mechanism,
  config storage, SSH-agent-vs-key-file auth fallback behavior, and a
  scoped re-check of whether SSH-key auth is still right for this
  specific client.
- **Out of scope:** Re-opening chat's platform-wide architecture (`006`),
  the admin/participant privilege model (`013`), the owner's own TUI
  client design, or `rebus`'s own backend implementation — none of those
  are reconsidered here. Actually creating the `github.com/monamaret/
  bateau` repo or writing its implementation code — this doc specifies
  what that repo must deliver, the work itself happens there.
- **Success criteria:** All three flagged follow-ups resolved (directly,
  or via owner decision) and recorded in this doc's Decision Log, then
  `005-private-chat-1to1.md` updated to match.
- **Non-goals:** Designing the owner's invite-issuing flow in detail
  (already resolved in `013` — admin actions live in the owner's
  TUI-embedded view); this doc is about `bateau`'s side of that exchange
  only.

## Background & Context

- **Relevant prior work:**
  - [006 → Open Questions](006-private-chat-application.md#open-questions) — the three follow-ups this doc resolves, verbatim.
  - [006 → Decision Log](006-private-chat-application.md#decision-log) — names this client `bateau` (`github.com/monamaret/bateau`) and the backend `rebus` (`github.com/monamaret/rebus`), each its own repo; `bateau` imports `rebus`'s published client package as an external Go module dependency, not an `internal/` package.
  - [002-hybrid-tui-layer.md → Security & Compliance](002-hybrid-tui-layer.md#security--compliance) — SSH-public-key-based provisioning, no IdP dependency, no long-lived credentials leave the client — the baseline this doc's auth re-check is measured against.
  - [013-cli-command-mapping-and-privilege-model.md → Decision Log](013-cli-command-mapping-and-privilege-model.md#decision-log) — confirms `bateau` has no admin role; this doc's command surface must not include any admin operation.
  - `github.com/monamaret/biblio`'s own distribution decision (already cited platform-wide) — GitHub Releases via `goreleaser` as the primary install path, `go install` as the alternative for those with a Go toolchain — the direct precedent for this doc's distribution question.
  - `github.com/monamaret/rook-reference`'s config-path convention (`$XDG_CONFIG_HOME/rook/config.json`, with an env var override) — the direct precedent for this doc's config-storage question.
- **Domain assumption:** The realistic guest for a private, terminal-
  based, SSH-key-authenticated chat app is a technically-inclined person
  (most likely another developer) who either already has an SSH keypair
  or can generate one trivially — this is not a general-consumer-facing
  product. This assumption directly shapes the auth-mechanism re-check
  below.
- **Stakeholders:** mona (the admin/owner) and whichever specific person
  is invited as a guest participant.

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `golang.org/x/crypto/ssh/agent` | Sign the auth challenge via a running SSH agent, if available | BSD-3-Clause | Mature, official Go extended-stdlib | Actively maintained by the Go team | High | Standard library for talking to an agent over `SSH_AUTH_SOCK` — already implicitly assumed by the platform-wide "SSH private key signs an auth challenge" pattern; this doc just makes the *fallback* behavior explicit. |
| `golang.org/x/crypto/ssh` (`ParsePrivateKeyWithPassphrase`/`ParseRawPrivateKeyWithPassphrase`) | Sign the challenge directly from a key file when no agent is available, prompting for a passphrase if the key is encrypted | BSD-3-Clause | Mature | Actively maintained by the Go team | High | Confirmed via Go's own package docs (see Web Citations): built-in support for both PEM-encrypted and OpenSSH-native encrypted private keys — no third-party dependency needed for the fallback path. |
| `golang.org/x/term` | Securely read a passphrase from the terminal (no echo) | BSD-3-Clause | Mature, official Go extended-stdlib | Actively maintained by the Go team | High | The standard, already-idiomatic way to prompt for a hidden passphrase in a Go CLI — no new dependency class introduced. |

**Selected:** Try the SSH agent first (if `SSH_AUTH_SOCK` is set and the
configured key is loaded); fall back to reading the private key file
directly, prompting for a passphrase via `golang.org/x/term` if it's
encrypted. This is a confirmed, real pattern (see Web Citations: "if
ssh-agent auth socket is present... used as primary authentication
method with fallback to private keys") — not a novel design, just
applying an established Go CLI convention to `bateau`.
**Rejected (with reason):** None — no credible alternative library was
needed; this is a small amount of glue code over the standard
`x/crypto/ssh` ecosystem already in use platform-wide.
**Open questions:** none for this section — resolved directly from the
established Go SSH-tooling pattern, not an owner preference call.

## Architecture Patterns

### Auth/session-state UX (resolves follow-up 1)

- **Resolution order**: (1) if `SSH_AUTH_SOCK` is set, query the agent
  for the configured identity's public key and request a signature from
  it; (2) if no agent, or the agent doesn't have the key loaded, read the
  private key file directly (`--identity`/`BATEAU_IDENTITY`, defaulting
  to `~/.ssh/id_ed25519`, mirroring `bbb-le`'s own `--identity`/`-i` flag
  convention already cited in this platform's research); (3) if the key
  file is encrypted, prompt for its passphrase via `golang.org/x/term`
  (no echo); (4) if no key is found at all, exit immediately with a
  clear, actionable error naming the path checked and the flag/env var to
  override it — never a generic "auth failed."
- This mirrors the exact fallback behavior already established as a
  real-world Go CLI pattern (see Web Citations) — not a new invention.

### Auth mechanism re-check (resolves follow-up 2)

- **SSH-key auth remains the right mechanism for `bateau` too — not a
  different one.** The original concern was that asking a non-technical
  guest to manage an SSH keypair is a real UX burden. But this platform's
  actual guest profile (per the domain assumption above) is a
  technically-inclined person for whom generating or already having an
  SSH key is a near-zero-friction step — and every alternative considered
  either reintroduces a second identity system (breaking the "one
  identity, no IdP" principle reinforced in `002`/`006`/`013`) or loses
  the durable, revocable identity an ongoing async conversation actually
  needs (a one-time link/ephemeral token has no story for "the guest
  comes back next week"). The genuine fix needed isn't a different
  mechanism, it's a **smoother bootstrap**: a documented one-liner
  (`ssh-keygen -t ed25519`) for a guest who doesn't already have a key,
  and a clear, copy-pasteable way to hand the resulting public key to the
  admin out-of-band (email, Signal, etc. — the invite *exchange* itself
  is explicitly outside any platform identity, by design).

### `bateau`'s full feature scope (resolves follow-up 3)

- **Distribution**: GitHub Releases via `goreleaser` from
  `github.com/monamaret/bateau` as the primary install path (prebuilt
  binaries), `go install github.com/monamaret/bateau` as the alternative
  for guests with a Go toolchain — directly reusing biblio's own
  already-established distribution precedent rather than inventing a new
  one.
- **Commands** (Cobra, mirroring bbb-le's CLI conventions —
  `<noun> <verb>` pattern, `--host`/`--identity` global flags, `--json`
  on list/history commands):
  - *(no subcommand)* — opens the interactive Bubble Tea TUI, the single
    conversation view (send/read/hide only — no admin operations, per
    `013`).
  - `bateau history [--limit N] [--since TIMESTAMP] [--json]` —
    non-interactive, scriptable message history (mirrors bbb-le's
    `dm history` shape).
  - `bateau send <message>` — non-interactive send, for scripting.
- **Config storage**: `$XDG_CONFIG_HOME/bateau/config.json` (or platform
  default `~/.config/bateau/...` fallback), with a `BATEAU_CONFIG` env
  var override — directly reusing rook-cli's own XDG config-path
  convention.
- **First-run flow**: on first launch with no config present, prompt for
  the identity path (or accept the `~/.ssh/id_ed25519` default), validate
  it can sign the auth challenge against `rebus` (failing clearly per the
  auth/session-state UX above if it can't), then perform an initial sync
  to populate the local cache before opening the TUI.

**Tradeoffs:** None significant beyond what's already accepted
platform-wide (SSH-key-based identity's setup cost vs. its IdP-free
durability) — this doc resolves UX/scope gaps, not a new tradeoff.
**Integration points:** `bateau`'s commands are thin wrappers over the
same public Go client package published from `rebus`
(`github.com/monamaret/rebus/client`, per `006`'s Decision Log) — no new
client-side logic beyond the auth fallback and the Cobra command surface
itself. `bateau` imports this as an external dependency, since `rebus`
and `bateau` are separate repos/modules.
**Data model implications:** none — this doc is entirely about the
client side; `rebus`'s backend schema (`013`'s `role` field, etc.) is
unaffected.

## Existing Codebase Evaluation

No `bateau` code exists yet (its repo, `github.com/monamaret/bateau`,
has not been created). `bbb-le`'s CLI conventions (`<noun> <verb>`,
`--host`/`--identity` globals, `--json` on list/history) and `rook-cli`'s
XDG config-path convention are the direct precedents reused above, both
already named in this platform's earlier research.

## Security & Compliance

- No long-lived credential is newly introduced — the agent-then-key-file
  fallback still never transmits the private key itself, consistent with
  the platform-wide posture already required.
- The passphrase prompt (when needed) happens entirely client-side via
  `golang.org/x/term` — never sent anywhere, never logged.

## Performance, Reliability & Operations

Not applicable beyond what's already covered in `006` — this doc is
about client UX/scope, not a new runtime characteristic.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| A guest with no existing SSH key finds the bootstrap step (`ssh-keygen`) confusing despite being "technical enough" by assumption | Low–Medium | Low | The first-run flow's error message (when no key is found) should name the exact command to run, not just state that no key was found | mona |
| The agent-fallback logic silently picks the wrong key if multiple identities are loaded in the agent | Low | Low–Medium | Match the agent's loaded keys against the specific public key registered for this guest (already known from the invite), not just "the first key the agent offers" | mona |

## Open Questions

None — all three follow-ups from `006` are resolved directly above, from
established Go SSH-tooling patterns and already-cited platform
precedents (biblio's distribution choice, rook-cli's config convention,
bbb-le's CLI shape), not an owner preference call.

## Decision Log

- **2026-06-24** — **Auth resolution order: SSH agent first (if
  `SSH_AUTH_SOCK` is set and the key is loaded), falling back to reading
  the private key file directly (prompting for a passphrase via
  `golang.org/x/term` if encrypted); a missing key exits with a clear,
  actionable error.** — Rationale: a confirmed, real-world Go CLI
  pattern, not a novel design; gives a graceful degradation path instead
  of an opaque auth failure. — Decided by: mona.
- **2026-06-24** — **SSH-key auth remains the correct mechanism for
  `bateau` — not replaced with a different one.** — Rationale: the
  realistic guest profile makes key setup low-friction; every alternative
  considered either breaks the platform's one-identity-system principle
  or loses durable, revocable identity. The actual fix is a smoother
  bootstrap, not a different mechanism. — Decided by: mona.
- **2026-06-24** — **`bateau`'s distribution: GitHub Releases via
  `goreleaser` (primary), `go install` (alternative), both from
  `github.com/monamaret/bateau`** — directly reusing biblio's own
  already-decided distribution precedent. — Decided by: mona.
- **2026-06-24** — **`bateau`'s commands: an interactive TUI by default,
  plus `history` and `send` non-interactive commands; config at
  `$XDG_CONFIG_HOME/bateau/config.json`** — directly reusing bbb-le's
  CLI shape and rook-cli's config-path convention. — Decided by: mona.

## Internal References

- [006-private-chat-application.md](006-private-chat-application.md) — the source of all three follow-ups this doc resolves, and the Decision Log entry naming `rebus`/`bateau` and their repos
- [005-private-chat-1to1.md (feature)](../features/005-private-chat-1to1.md) — the feature item this research will refine
- [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) — confirms `bateau` has no admin role, bounding this doc's command surface
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — the SSH-key-based, IdP-free identity baseline this doc's auth re-check is measured against
- `github.com/monamaret/biblio` — distribution precedent (GitHub Releases via `goreleaser`, `go install` alternative)
- `github.com/monamaret/rook-reference`, `github.com/monamaret/bbb-le` — config-path and CLI-shape precedents
- `github.com/monamaret/rebus` (not yet created) — publishes the client package `bateau` imports
- `github.com/monamaret/bateau` (not yet created) — this doc's subject; full implementation lives there
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and the Security & Operational Posture section, both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `golang.org/x/crypto/ssh/agent` package docs | https://pkg.go.dev/golang.org/x/crypto/ssh/agent | 2026-06-24 | Confirms the standard library for SSH-agent-based signing this doc's primary auth path uses. |
| Web search — Go SSH agent fallback to private key file pattern | (search query, no single URL) | 2026-06-24 | Confirms the agent-first, key-file-fallback pattern as an established real-world Go CLI convention ("if ssh-agent auth socket is present... used as primary authentication method with fallback to private keys"), and that `golang.org/x/crypto/ssh` natively supports both PEM- and OpenSSH-encrypted private keys via `ParsePrivateKeyWithPassphrase`/`ParseRawPrivateKeyWithPassphrase` — basis for needing no third-party dependency for the fallback path. |

## Appendix

None yet.
