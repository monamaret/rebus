# Research: WebAuthn implementation mechanics — library and credential storage

**Status:** Complete — library and schema resolved directly from go-webauthn's own current maintenance status and documented interface; ready to update specs/research/002-hybrid-tui-layer.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) settled WebAuthn's
*policy* shape in detail — one identity with two credential types (SSH
key for TUI, WebAuthn/passkey for web), enrollment-link bootstrap reusing
the existing admin-invite command, httpOnly-cookie session tokens — and
this has been cited as "the decided mechanism" across `004`, `005`,
`006`, and `011` ever since. None of those docs ever named a concrete Go
library or a credential-storage schema, the same kind of black box
`010`/`011`/`012` each resolved for their own topics. [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md)'s
MCP adapter literally enforces "the access-controlled-by-default WebAuthn
check" — a check this doc is what actually defines the mechanics of.

- **Goal:** Select a Go WebAuthn library and define the concrete
  credential-storage schema and registration/auth flow, narrow enough to
  update `002` and every doc that cites its WebAuthn decision.
- **In scope:** WebAuthn server library selection; the `User`/credential
  interface implementation against the existing owner-controlled identity
  registry; the Firestore schema for stored credentials; how this
  integrates with the already-decided enrollment-link bootstrap and
  session-token mechanics.
- **Out of scope:** Re-opening `002`'s settled policy decisions (one
  identity/two credential types, enrollment-link bootstrap, httpOnly
  session cookies) — not reconsidered here.
- **Success criteria:** A confirmed library and schema, recorded in this
  doc's Decision Log.
- **Non-goals:** Building the actual enrollment-link UI/email-or-message
  delivery mechanism — that's implementation detail for whichever
  feature first exercises live web mode, not a research blocker here.

## Background & Context

- **Relevant prior work:**
  - [002 → Security & Compliance](002-hybrid-tui-layer.md#security--compliance) — the full policy shape this doc implements concretely: one identity/two credential types, enrollment-link bootstrap via the existing admin-invite command, httpOnly/secure session cookies refreshed via re-auth.
  - [011 → Architecture Patterns](011-shared-operation-definitions-and-mcp-sdk.md#architecture-patterns) — the MCP adapter's access-control check this doc's library/flow choice is what concretely backs.
  - [005-generic-app-scaffold.md → Decision Log](005-generic-app-scaffold.md#decision-log) — MCP defaults to access-controlled, reusing this WebAuthn decision; live-mode web instances are where this doc's mechanism is actually exercised.
- **Domain assumption:** No app in the current release actually exercises
  live web mode yet (the dashboard is static-only, chat's web client is
  deferred) — this doc's findings apply to whichever app first needs
  live mode, the same scoping `012` already used for its own
  not-yet-exercised live-mode findings.
- **Stakeholders:** mona (the only registered identity today) and any
  future outside-domain web-client user.

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **`go-webauthn/webauthn`** | WebAuthn/FIDO2 server-side registration and assertion ceremonies | BSD-3-Clause | Mature, conformance-tested against the official WebAuthn conformance tools | Actively maintained — the community-maintained successor to `duo-labs/webauthn` | High | Confirmed via the project's own docs and community guidance (see Web Citations): projects on the original `duo-labs/webauthn` are told to migrate to this fork, which is the currently maintained Go WebAuthn server implementation. |
| `duo-labs/webauthn` | Same purpose, original implementation | BSD-3-Clause | Was mature | **Effectively unmaintained** — superseded by `go-webauthn/webauthn` | Rejected | Confirmed via the same source: explicitly told to migrate away from this one. Choosing it now would mean adopting a library already pointing users elsewhere. |
| Rolling a custom WebAuthn implementation | Same purpose, in-house | N/A | N/A | N/A | Rejected | WebAuthn's ceremony/attestation-verification logic is exactly the kind of well-specified, security-critical protocol implementation Principle I's minimal-dependency bias does *not* apply to reimplementing — a maintained, conformance-tested library is the correct call, not a dependency to avoid. |

**Selected:** `go-webauthn/webauthn` for registration/assertion ceremonies
on the backend.
**Rejected (with reason):** `duo-labs/webauthn` (unmaintained, library
itself points elsewhere); a custom implementation (wrong place to apply
minimal-dependency bias). See table.
**Open questions:** none for this section — resolved directly from the
library's own current maintenance status, not an owner preference call.

## Architecture Patterns

- **The existing owner-controlled identity registry implements
  `go-webauthn`'s required `User` interface directly — no second user
  store.** Per the library's own interface (`WebAuthnID`, `WebAuthnName`,
  `WebAuthnCredentials`, confirmed via its own package docs, see Web
  Citations), the same Firestore-backed identity record that already
  holds a person's registered SSH public key (per `002`'s "one identity,
  two credential types") gains a `webauthnCredentials` array field,
  populated and read through these three interface methods — not a
  parallel user table.
- **Credential storage schema**, per credential, mirroring the library's
  own required `Credential`/`Authenticator` fields (confirmed via its own
  package docs, see Web Citations): `id` (the credential ID, the primary
  lookup key at authentication time), `publicKey` (CBOR-encoded COSE
  key), `attestationType`/`attestationFormat`, `transport` (hints), and
  an `authenticator` sub-object holding `aaguid`, `signCount`, and
  `cloneWarning`. `signCount` and `cloneWarning` **must be re-written on
  every successful login** (per the library's own documented
  requirement) — this is a write, not just a read, on the auth path.
- **Registration ceremony**: `BeginRegistration` → the browser's
  `navigator.credentials.create()` → `FinishRegistration`, triggered the
  first time a person follows their enrollment link (the link itself
  already decided in `002` — this doc only adds the concrete ceremony
  underneath it).
- **Authentication ceremony**: `BeginLogin` → `navigator.credentials.get()`
  → `FinishLogin`, issuing the short-lived, httpOnly/secure session
  cookie `002` already decided, with the `signCount` write-back above
  happening as part of `FinishLogin`'s success path.
- **Relying Party ID partitioning**: per the library's own storage
  guidance (confirmed via Web Citations: credentials "must be
  partitioned by RP ID"), credentials are scoped to whichever domain
  the live-mode web instance is actually served from — relevant once a
  real domain exists for the first live-mode app, not designed further
  here.

**Tradeoffs:** None beyond what's already accepted in `002` — this doc
adds the concrete mechanics under an already-settled policy, not a new
tradeoff of its own.
**Integration points:** Directly backs `011`'s MCP-adapter access-control
check and `005`'s "MCP defaults to access-controlled" decision — no new
enforcement layer, just the concrete implementation of the check both
already assume exists.
**Data model implications:** The identity registry's existing Firestore
schema gains one array field (`webauthnCredentials`) per user record —
no new collection, no new service.

## Existing Codebase Evaluation

No `skipper` WebAuthn code exists yet, and no reference repo (biblio,
bbb-le, rook-reference, sight) implements WebAuthn — `002`'s own policy
design and `go-webauthn`'s documented interface are the only direct
precedents here, not an adapted reference-repo pattern.

## Security & Compliance

- `signCount`/`cloneWarning` tracking (per the library's own design) is
  the standard WebAuthn clone-detection mechanism — if a future
  authentication's `signCount` doesn't increase as expected, that's a
  signal of a possibly-cloned authenticator; this doc doesn't design a
  bespoke response to that signal beyond noting it's available, per
  Non-goals.
- Raw attestation data (`CredentialAttestation`'s `ClientDataJSON`,
  `AuthenticatorData`, etc.) must be persisted byte-for-byte per the
  library's own requirement, to support later re-verification via
  `Credential.Verify()` — stored alongside the rest of the credential
  record, not discarded after initial registration.
- No new credential *type* beyond what `002` already decided (WebAuthn/
  passkeys) — this doc is the mechanics under that decision, not an
  expansion of it.

## Performance, Reliability & Operations

- WebAuthn ceremonies are inherently interactive, user-present operations
  (a registration or login click) — no background/polling characteristic
  to design around, consistent with this platform's manual-trigger
  posture elsewhere.
- The `signCount` write-back on every login is a small, synchronous
  Firestore write on the auth path — negligible at this platform's
  single-owner-plus-occasional-guest scale.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Forgetting to write back `signCount`/`cloneWarning` after a successful login, silently breaking clone detection over time | Low–Medium | Medium | Document this as a required step of `FinishLogin`'s success path, not an optional follow-up — per the library's own explicit requirement | mona |
| Treating the identity registry's `webauthnCredentials` field as fully optional/sparse without considering the multi-device case (a person registering more than one authenticator) | Low | Low | The field is already an array, not a single object — multiple credentials per user is the default shape, not a later addition | mona |

## Open Questions

None — the library choice and schema are resolved directly from
`go-webauthn`'s own current maintenance status and documented interface
requirements, not an owner preference call.

## Decision Log

- **2026-06-24** — **`go-webauthn/webauthn` is the WebAuthn server
  library — not `duo-labs/webauthn` (unmaintained, superseded by this
  same fork) and not a custom implementation.** — Rationale: actively
  maintained, conformance-tested; WebAuthn's protocol complexity is
  exactly the kind of security-critical logic Principle I's
  minimal-dependency bias doesn't apply to reimplementing. — Decided by:
  mona.
- **2026-06-24** — **The existing owner-controlled identity registry
  implements `go-webauthn`'s `User` interface directly (gaining a
  `webauthnCredentials` array field); no second/parallel user store is
  introduced.** — Rationale: keeps one identity record per person, per
  `002`'s already-decided "one identity, two credential types" shape. —
  Decided by: mona.
- **2026-06-24** — **Credential storage mirrors the library's own
  required `Credential`/`Authenticator` fields exactly** (`id`,
  `publicKey`, attestation metadata, `aaguid`, `signCount`,
  `cloneWarning`), with `signCount`/`cloneWarning` re-written on every
  successful login per the library's documented requirement. — Decided
  by: mona.

## Internal References

- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — the WebAuthn policy decision this doc implements concretely
- [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) — the MCP-adapter access-control check this doc's mechanism backs
- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — the access-controlled-by-default decision this doc's mechanism implements
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias, correctly *not* applied to reimplementing WebAuthn) and Principle VI (secure-by-default), both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `go-webauthn/webauthn` — GitHub repo and package docs | https://github.com/go-webauthn/webauthn ; https://pkg.go.dev/github.com/go-webauthn/webauthn/webauthn | 2026-06-24 | Confirms this is the actively maintained Go WebAuthn server library, the community-maintained successor to `duo-labs/webauthn` — basis for selecting it over the original, now-superseded library. |
| Web search — "go-webauthn/webauthn library Go WebAuthn server implementation maintained" | (search query, no single URL) | 2026-06-24 | Confirms projects on `duo-labs/webauthn` are explicitly directed to migrate to `go-webauthn/webauthn` — basis for rejecting the original library. |
| `pkg.go.dev/github.com/go-webauthn/webauthn/webauthn` — `User`/`Credential`/`Authenticator` types | https://pkg.go.dev/github.com/go-webauthn/webauthn/webauthn | 2026-06-24 | Source of the exact `User` interface methods (`WebAuthnID`/`WebAuthnName`/`WebAuthnCredentials`) and `Credential`/`Authenticator` struct fields (`ID`, `PublicKey`, `AAGUID`, `SignCount`, `CloneWarning`, attestation data) this doc's storage schema is built from, plus the documented requirement to write `SignCount`/`CloneWarning` back on every login and partition credentials by Relying Party ID. |

## Appendix

None yet.
