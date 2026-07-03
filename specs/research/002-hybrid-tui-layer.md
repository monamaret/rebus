# Research: Hybrid TUI layer — usage categories and interaction-model reference patterns

**Status:** Complete — all open questions resolved; ready for feature-item creation
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

`skipper`'s hybrid TUI layer (constitution Principle VIII) runs primarily
client-side, with lazy data sync against the backend for specific pages.
`github.com/monamaret/bbb-le` was the first reference cited for this layer.
Two more reference repos — `github.com/monamaret/rook-reference` and
`github.com/monamaret/sight` — have since been identified as useful,
described as "stateless TUI" where "SSH is used elsewhere in the auth
flow." Reviewing all three together surfaces that they are **not the same
interaction model**: bbb-le serves an interactive Bubble Tea application
*inside* the SSH session itself, while rook-reference and sight use SSH
only to sign/validate an auth challenge and otherwise run a stateless
client (local-first TUI) talking to an RPC/HTTPS server.

The platform's actual TUI usage spans (at least) four distinct categories,
per the owner's description:

1. **Dev-artifact / context work** — produces artifacts that need to live
   in source control, or that get pulled in as context for later use.
2. **Social** — a private chat application accessed through the terminal,
   with a secure mechanism for provisioning access to users *outside the
   domain* (i.e. not gated by a corporate/Google identity).
3. **Development-toolchain integration** — work that should integrate with
   existing development toolchains and infrastructure in GitHub.
4. **TUI-only backend apps** — apps deployed on Kubernetes that are
   reachable *only* over the TUI/SSH surface, with no other access path
   (the named example is `github.com/charmbracelet/soft-serve`, a
   self-hosted Git server whose primary interface is SSH).

- **Goal:** Determine which interaction model (or models) applies to each
  category, and what `skipper` needs to scaffold/standardize across them.
- **In scope:** Comparing bbb-le's SSH-session-served-TUI model against
  rook/sight's stateless-client + RPC-over-SSH-authenticated model; mapping
  each of the four usage categories onto a model; identifying the
  provisioning/auth pattern for users outside the domain.
- **Out of scope:** Copying any reference repo's code directly; deciding
  exact RPC wire formats or TUI screen layouts.
- **Success criteria:** A category → interaction-model mapping recorded in
  this doc's Decision Log, narrow enough to drive a feature item.
- **Non-goals:** Building a single one-size-fits-all TUI framework if the
  categories genuinely warrant different models — per constitution
  Principle I, don't force speculative unification.

## Background & Context

- **Relevant prior work:** `specs/research/architecture.md` already names
  bbb-le as the TUI layer's reference. `specs/research/roadmap.md` §0
  already flags "hybrid TUI layer stack specifics" as an open question this
  doc narrows.
- **`github.com/charmbracelet/soft-serve`** (external, not owned by mona) —
  cited by name as the shape category 4 should follow: a Kubernetes-hosted
  service whose only interface is an SSH-served TUI/CLI, with no separate
  web or RPC surface exposed.
- **Domain assumptions:** SSH public-key auth (no passwords) is already the
  shared baseline across bbb-le, rook-reference, and sight — all three
  authenticate by registered public key, not by a GCP/Google identity. This
  is what makes "provisioning access to users outside the domain" tractable
  in all three reference repos: identity is the SSH key, not an IdP account.
- **Stakeholders:** mona (sole owner/operator; also the admin for any
  outside-domain users invited into the social/chat surface).

## Libraries & Stack

| Repo | Interaction model | Server-side TUI? | SSH's role | Sync model |
| --- | --- | --- | --- | --- |
| `bbb-le` | SSH session *is* the app | Yes — Wish multiplexes a server-side Bubble Tea app (chat TUI, profile TUI) over the SSH session | Transport **and** session — the SSH connection stays open for the interactive session | N/A — server is stateful, session-resident (Charm KV/BadgerDB) |
| `rook-reference` (`rook-cli` + `rook-server`) | Stateless client + RPC/HTTPS server | No — `rook-cli` is a local-first Bubble Tea TUI that runs entirely on the user's machine | Auth only — the user's SSH private key signs a challenge locally (`POST /auth/verify`); the key never leaves the CLI; no SSH transport to the server at all (HTTPS + gRPC instead) | Explicit pull-sync: stash/messages/guides sync on demand over HTTPS; local flat-file store (`$XDG_CONFIG_HOME/rook/storage/`) is the offline source of truth |
| `sight` (`sightd` + `sight-cli`) | Stateless client + RPC server | No — as of v0.8.2 (`specs/009-server-interface-prep`), bubbletea/bubbles/lipgloss were *removed* from the server; an `interactiveRejectionMiddleware` actively rejects interactive SSH sessions | Transport for the RPC channel + auth (`charmbracelet/ssh` + `wish`), but no interactive session is ever rendered server-side | Explicit incremental sync: `NoteService.SyncNotes(ctx, customerID, since time.Time)` — "returns all notes modified at or after `since`... used by the RPC sync path to push incremental updates to clients" |

**Selected:** *(pending Decision Log — see Architecture Patterns below for
the candidate per-category mapping)*
**Rejected (with reason):** Treating "hybrid TUI layer" as a single
interaction model when the reference repos themselves demonstrate two
genuinely different, deliberately-chosen models (sight explicitly *removed*
server-side TUI rendering in favor of the rook-style stateless model — that
migration is itself evidence the two models aren't interchangeable
defaults).
**Open questions:** see below.

## Architecture Patterns

Confirmed model per usage category (see Decision Log):

- **1. Dev-artifact / context work → stateless client + local Markdown
  stash, rook pattern (not sight's context-engine/entity-search model).**
  Artifacts are source-controlled `.md` files, synced like rook-cli's
  `stash/<space-id>/` pull-sync — the same shape `specs/research/*.md`
  already uses for this exact purpose (source-controlled, read back in as
  context later). A context-engine/entity-search service is explicitly
  deferred: revisit only if local full-text search genuinely stops being
  enough, not speculatively.
- **2. Social / private chat with outside-domain provisioning → general
  default is the bbb-le pattern (SSH session is the app), with a simple
  custom chat protocol (not an embedded IRC server) — but the first
  release's concrete chat app is a scoped exception, see below.** For a
  multi-user/group, terminal-only chat surface, the session-is-the-app
  model still fits best, with Ergo/IRC protocol compliance dropped (the
  only client would be `skipper`'s own bbb-le-style TUI, so IRC-client
  interoperability buys nothing under Principle I). **However, the first
  release's chat app is explicitly 1:1 and stateless, with both a TUI and
  a web client** — a web client is structurally impossible under
  SSH-session-multiplexing, so this app uses the **rook/sight pattern**
  instead (stateless client + RPC/HTTPS backend, e.g. Firestore-backed
  message store; two thin clients sharing one API). The "outside the
  domain" provisioning requirement still maps to bbb-le's
  `admin invite <public-key-path>` / rook-server-cli's `user register-key`
  pattern (SSH-public-key-based, no IdP dependency) regardless of which
  interaction model the chat app itself uses — provisioning is orthogonal
  to which pattern serves the chat traffic.
- **3. Development-toolchain / GitHub integration → stateless client
  calling GitHub's API directly, cached locally — no custom backend at
  all.** GitHub *is* the backend here; `skipper` doesn't operate an RPC
  service for this category. The TUI calls GitHub's REST/GraphQL API (or
  wraps `gh`) and caches results with the same local-cache-with-fallback
  discipline as the app catalog. A small cross-repo aggregator (e.g.
  unified PR/CI status across every skipper-managed repo, since each lives
  in its own repo per Principle VIII) is an explicitly deferred escalation,
  built only if that concrete cross-repo need actually shows up.
- **4. TUI-only Kubernetes apps → bbb-le/Soft Serve pattern, one
  SSH-served binary per app, with the app catalog (not a live gateway)
  supplying the connect address.** Matches bbb-le's existing shape
  directly (a Go binary, deployed on GKE via Kustomize base + overlays,
  serving its entire interface over Wish/SSH). No Wishlist-style gateway
  process is built — the already-decided app catalog (durable static
  artifact + client-side cache) already does the "what apps exist, where
  do I connect" job a gateway would otherwise do, so adding a live gateway
  on top would be a redundant second discovery mechanism.

**Confirmed split:** Categories 1 and 3 share the stateless-client pattern
(local cache + either local files or a remote-but-already-managed API —
GitHub's — never a `skipper`-operated backend service for either);
categories 2 and 4 share the SSH-session-served pattern (bbb-le/Soft
Serve). This is a clean 2-way split, and in both halves the "remote state"
is either something `skipper` doesn't have to build (GitHub) or something
already decided (the app catalog) — no new always-on service is introduced
anywhere in this layer.
**Integration points:** category 3's local API cache and the app catalog's
client-side cache (already decided) are the same caching discipline reused
twice — likely worth a single shared "cached remote lookup" helper in
`skipper`'s TUI client code rather than two separate implementations.
**Data model implications:** none yet — no `skipper`-side TUI scaffolding
code exists.

### App catalog architecture (single interface over many independent apps)

Regardless of how the four categories ultimately split across the two
interaction models, presenting them through **one TUI front door (and
eventually one web dashboard) over many independently-deployed apps**
needs a discovery mechanism: a catalog of "what apps exist, and which
driver/endpoint reaches each one." Two requirements drove this design,
both raised by the owner directly:

1. Adding or removing an app must not require redeploying the TUI gateway
   (or the web dashboard) — the app set changes far more often than the
   front door's own code.
2. The catalog must not be a single point of failure, and specifically
   must not require a *live* connection to function for the common case
   (browsing/launching apps already known to the client) — even though
   this platform has exactly one user, an always-on stateful "registry
   service" is an unnecessary new failure mode to own.

**Decided shape** (see Decision Log):

- **Source of truth = a durable, static artifact, not a running service.**
  A small `catalog.json`-shaped document (or equivalent) stored in Cloud
  Storage or Firestore — both GCP-managed, high-read-availability
  primitives per constitution Principle V — listing each app's name,
  interaction model (SSH-session-served vs. stateless RPC), and connection
  info. Updated only when an app is added/removed/moved, via a small admin
  command — never read-path-coupled to that command's uptime. This
  directly avoids building and operating a bespoke always-on registry
  process, which would otherwise be the platform's actual single point of
  failure.
- **Client-side cache with fallback, rook-style.** Every client (the TUI
  gateway, and later any web dashboard) mirrors `rook-cli`'s
  `cache/spaces.json` "app ACL cache" pattern exactly: fetch the catalog
  opportunistically (on launch, or on a background refresh), always render
  from the last-successfully-fetched local copy, and fall back to that
  cached copy transparently if the GCS/Firestore read fails or is slow.
  The only thing that degrades when the catalog source is unreachable is
  *discoverability of changes* (a brand-new app won't show up until the
  next successful refresh) — every app already known to the client stays
  fully launchable, since launching only needs that app's own endpoint to
  be up, not the catalog's.
- **Net effect:** no live dependency for the common path at all. The two
  remaining failure modes — GCS/Firestore having an outage, or the local
  cache being empty on a brand-new client with no network — are both rare
  and gracefully degrading (stale-but-usable, or "first run, must be
  online once") rather than "platform unusable."

## Existing Codebase Evaluation

`skipper` has no TUI-layer scaffolding code yet — greenfield, building on
patterns from bbb-le, rook-reference, and sight (all external reference
repos, not vendored into `skipper`).

## Security & Compliance

- **Outside-domain provisioning is SSH-public-key-based, not IdP-based**,
  across all three reference repos — this is the load-bearing mechanism
  that makes inviting users outside the developer's own domain/org
  tractable at all, since it sidesteps needing those users in GCP IAM,
  Google Workspace, or any other org-scoped identity system.
- **Admin role is backend-only and separate from application access**
  (bbb-le's documented role split) — the person who can invite/revoke keys
  is not automatically granted chat/app access by virtue of that role. This
  is a useful default to carry into `skipper`'s own admin tooling for the
  social/chat category.
- **Web clients authenticate via WebAuthn/passkeys, not the SSH key
  directly.** Browsers can't hold/use an SSH private key the way a CLI
  can, so any `skipper`-scaffolded web client (e.g. the chat app's web UI)
  registers a separate WebAuthn credential per device/browser, tied to the
  *same* owner-controlled user record that holds the SSH key for TUI
  access — one identity, two credential types, both still public-key-based
  and IdP-free. First-device enrollment is bootstrapped via a one-time,
  short-lived enrollment link issued through the same admin invite command
  already used for SSH-key registration; every login after that uses the
  passkey directly (no email/password, no third-party OAuth). Post-auth
  sessions use a short-lived signed token in an httpOnly/secure cookie,
  refreshed via re-auth rather than a long-lived stored credential. See
  Decision Log. *(Mechanism filled in — see
  [020-webauthn-implementation-mechanics.md → Decision Log](020-webauthn-implementation-mechanics.md#decision-log):
  **`go-webauthn/webauthn`** is the server library (the actively
  maintained successor to the now-superseded `duo-labs/webauthn`); the
  owner-controlled identity registry implements the library's `User`
  interface directly — no second user store — gaining a
  `webauthnCredentials` array field per record; credential storage
  mirrors the library's own required fields (`id`, `publicKey`,
  attestation metadata, `aaguid`, `signCount`, `cloneWarning`), with
  `signCount`/`cloneWarning` re-written on every successful login per the
  library's documented clone-detection requirement.)*
- **No long-lived credentials leave the client.** rook-cli signs the auth
  challenge with the user's SSH private key locally and never transmits
  the key itself; session tokens are held in-memory only and discarded on
  process exit. This aligns with constitution Principle VI (secure-by-
  default, least privilege) and should be the default expectation for any
  `skipper`-scaffolded TUI client, regardless of which interaction model it
  uses.
- **Key revocation** needs an explicit, deliberate flow per category (bbb-le
  has `admin revoke-key`; rook's admin path is `rook-server-cli`) — this is
  a concrete capability `skipper`'s own admin tooling needs to scaffold for
  whichever app(s) end up in the social/chat and TUI-only-app categories.

## Performance, Reliability & Operations

Not yet applicable — no implementation exists. Sync latency/volume
characteristics (categories 1 and 3) and SSH-session resource limits
(categories 2 and 4, relevant to GKE pod sizing) become concrete once a
feature item exists.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Treating bbb-le as *the* TUI reference (as the original architecture note did) and missing that categories 1/3 actually fit the rook/sight model better | Medium | Medium | This doc records the split explicitly; update `architecture.md` once categories are confirmed | mona |
| Outside-domain key provisioning becomes a backdoor if admin-invite tooling isn't itself access-controlled | Low–Medium | High | Carry over bbb-le's backend-only admin role split; require explicit revocation tooling from day one, not as a follow-up | mona |
| Building two interaction-model scaffolds (stateless-RPC vs. SSH-session-served) doubles `skipper`'s TUI scaffolding surface | Medium | Medium | Confirm the 2-pattern split (not 4 bespoke ones) before implementation; reuse rook/sight's pattern for 1+3 and bbb-le/Soft Serve's for 2+4 | mona |
| An always-on, custom registry/catalog service becomes the platform's real single point of failure, even though individual apps are independently deployable | Medium | Medium | Source catalog from a durable static artifact (GCS/Firestore), not a bespoke service process; clients cache the last-known-good catalog locally and fall back to it (rook-style), so no live read is required for the common path | mona |

## Open Questions

All five questions originally listed here are resolved — see the Decision
Log. None of the four categories' interaction models, the catalog
architecture, or the chat/context storage shapes remain open at the
research-doc level. Remaining work is implementation-scoping
(feature item creation), not further research, *except*:

- [ ] Exact shape of the optional category-3 cross-repo aggregator
  (deferred escalation — only scope this if/when a real cross-repo need
  shows up, per its Decision Log entry) — owner: mona
- [ ] Exact shared "cached remote lookup" helper design noted under
  Architecture Patterns → Integration points (reused by the catalog client
  and the GitHub-API client) — implementation detail, not a research
  blocker.

## Decision Log

- **2026-06-24** — `skipper`'s single TUI front door (and, later, any web
  dashboard) discovers apps via an **app catalog stored as a durable static
  artifact** (Cloud Storage or Firestore — GCP-managed, high-read-
  availability primitives), not a bespoke always-on registry service.
  Every client mirrors `rook-cli`'s local cache-with-fallback pattern
  (`cache/spaces.json`): render from the last-fetched local copy, refresh
  opportunistically, and fall back to the cached copy if the catalog source
  is unreachable. Adding/removing an app is a catalog write (via a small
  admin command) plus deploying that app's own manifest — it never requires
  redeploying the TUI gateway or web dashboard. — Rationale: removes the
  single point of failure a custom registry process would otherwise be,
  without sacrificing the "one interface over many independently
  deployable/maintainable apps" goal; reuses a pattern already proven in
  `rook-reference` rather than inventing a new one. — Decided by: mona.

- **2026-06-24** — **Category 1 (dev-artifact/context work) uses a local
  Markdown stash (rook pattern), not a context-engine/entity-search service
  (sight pattern).** Source-controlled `.md` files, pull-synced like
  rook-cli's `stash/<space-id>/`. — Rationale: matches how
  `specs/research/*.md` itself already works for "artifacts pulled in as
  context later"; a context-engine service is real infrastructure
  investment unjustified for one person's notes absent a demonstrated
  search-quality gap (Principle I). Escalate later only if local full-text
  search genuinely stops being enough. — Decided by: mona.
- **2026-06-24** — **Category 2 (private chat) uses a simple custom
  message/channel protocol, not an embedded IRC server (Ergo).** ~~The
  SSH-session-served model (bbb-le pattern) stays; only the protocol
  underneath changes.~~ — **Superseded 2026-06-24, see below — the
  interaction-model half of this decision changed; the
  "no embedded IRC, simple custom protocol" half still holds.**
  — Decided by: mona.
- **2026-06-24 (supersedes part of the entry above)** — **The first
  release's chat app (1:1, stateless) uses the rook/sight pattern
  (stateless client + RPC/HTTPS backend), not the bbb-le SSH-session-served
  pattern, and ships both a TUI client and a web client against the same
  backend.** — Rationale: the concrete first-release spec calls for a
  stateless, 1:1 chat app with **both a TUI and a web UI** — a web UI is
  structurally impossible under SSH-session-multiplexing (Wish only serves
  terminal clients, never a browser), so the bbb-le pattern cannot satisfy
  this requirement regardless of preference. The earlier category-2 mapping
  assumed a multi-user/group, terminal-only chat shape (closer to bbb-le's
  own IRC-style design goals); the actual first-release shape is narrower
  (1:1) and explicitly cross-surface, which the rook/sight pattern fits
  directly: one stateless backend (e.g. Firestore-backed message store),
  two thin clients. The "simple custom protocol, not embedded IRC" part of
  the original decision is unaffected and still applies. This narrows but
  does not invalidate category 2 generally — a *future*, multi-user/group,
  terminal-only chat surface could still reasonably use the bbb-le pattern;
  this decision is scoped to the first release's 1:1 chat app specifically.
  — Decided by: mona.
- **2026-06-24 (amends the entry above)** — **The web (and mobile) client
  for chat is deferred out of this release's scope; v1 ships TUI-only.**
  The rook/sight stateless-pattern decision above is unaffected — chat
  stays a stateless RPC/HTTPS backend, it just has exactly one client
  (TUI) for now instead of two. The dual-thin-client idea (CLI + web
  against one stateless backend) is generalized into its own reusable
  scaffold instead of being chat-specific — see
  [005-generic-app-scaffold.md](005-generic-app-scaffold.md). When/if chat
  gets a web client later, it can reuse that scaffold and the WebAuthn
  auth decision below rather than building bespoke web-client plumbing
  just for chat. — Decided by: mona.
- **2026-06-24** — **Web clients (starting with the chat app's web UI) use
  WebAuthn/passkeys, registered into the same owner-controlled identity
  registry as SSH keys, with a one-time enrollment link for first-device
  bootstrap.** Rejected alternatives: device-linking/pairing from an
  already-authenticated TUI session (rejected because it requires TUI
  access to bootstrap a web-only guest, which is too restrictive for an
  outside-domain participant who may never use the TUI); magic-link/email
  login (rejected — introduces a second, weaker parallel identity system);
  OAuth/social login (rejected — reintroduces the IdP dependency the
  SSH-key model exists to avoid for outside-domain users); username/
  password (rejected — reintroduces credential-storage/reset overhead
  inconsistent with the platform's passwordless posture everywhere else).
  — Rationale: WebAuthn is still public-key-based and IdP-free, has native
  browser/OS support (no extension or native app required), and extends
  the *same* identity registry already used for SSH keys rather than
  building a second one — one identity, two credential types. — Decided
  by: mona.
- **2026-06-24** — **Category 3 (GitHub/dev-toolchain integration) calls
  GitHub's API directly (or wraps `gh`), cached locally — `skipper` does
  not operate a custom backend service for this category.** A cross-repo
  aggregator (unified PR/CI status across all skipper-managed repos) is an
  explicitly deferred escalation, scoped only if that concrete need
  actually shows up. — Rationale: GitHub already is the backend; building
  a redundant one would violate Principle I with no offsetting benefit.
  — Decided by: mona.
- **2026-06-24** — **Category 4 (TUI-only Kubernetes apps) uses one
  SSH-served binary per app (bbb-le/Soft Serve pattern); no separate
  Wishlist-style gateway is built.** The already-decided app catalog
  supplies the per-app connect address, doing the discovery job a gateway
  would otherwise duplicate. — Rationale: avoids introducing a second,
  redundant live discovery mechanism alongside the catalog. — Decided by:
  mona. *(Amended 2026-06-24, see [007-tui-shell-dispatch-and-catalog.md → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log):
  this rejection targeted **building** a custom discovery gateway that
  would duplicate the catalog's job. It does not preclude **using** the
  real `github.com/charmbracelet/wishlist` tool as a shared
  network-multiplexing layer for Wish-pattern SSH apps — a distinct
  problem (one external endpoint for many SSH backends) from app
  discovery, which the catalog continues to own exclusively. Category 4
  apps now sit behind one shared Wishlist instance instead of one
  LoadBalancer Service each.)*
- **2026-06-24** — **Confirmed: categories 1+3 share the stateless-client
  caching pattern (local files or an already-managed remote API — never a
  `skipper`-operated backend) and categories 2+4 share the
  SSH-session-served pattern (bbb-le/Soft Serve).** This is the final,
  clean 2-way interaction-model split for the hybrid TUI layer **as a
  general default**. — Decided by: mona. *(Caveat added 2026-06-24: the
  first release's specific chat app is an explicit, scoped exception —
  see the supersession entry above. The 2-way split is the default to
  reach for; a given app's concrete requirements, like needing a web
  client, can still override it.)* *(Amended 2026-06-24, see
  [007-tui-shell-dispatch-and-catalog.md → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log):
  this 2-way split describes **interaction-model categories**, not a 1:1
  mapping to shell-level **dispatch mechanisms**. The stateless-client
  category (1+3) is served by *two* distinct shell dispatch mechanisms —
  in-process embedded views (stash, chat) and CLI-exec (the generic
  scaffold's dashboard) — while the SSH-session-served category (2+4) is
  served by one (SSH-passthrough, via Wishlist). So the shell implements
  three dispatch mechanisms total, grouped under these two categories,
  not three categories.)*

> **Format note (2026-07-03):** this doc predated the current
> `research-template.md`; its single `## References` list has been split into
> `## Internal References` + `## Web Citations` to match the template.

## Internal References

- [architecture.md](architecture.md) — confirms TUI layer = hybrid
  client-side + lazy backend sync; bbb-le originally named as reference
- [roadmap.md §0](roadmap.md#0-outstanding-design-questions) — "hybrid TUI
  layer stack specifics" open item this doc narrows
- [020-webauthn-implementation-mechanics.md](020-webauthn-implementation-mechanics.md) — the concrete WebAuthn library and credential-storage schema implementing this doc's policy decision

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `monamaret/bbb-le` | https://github.com/monamaret/bbb-le | 2026-06-24 | SSH-session-served TUI reference (Bubble Tea, Wish, Ergo IRC, Charm identity/KV, GKE via Kustomize) — the category 2/4 interaction-model precedent. |
| `monamaret/rook-reference` | https://github.com/monamaret/rook-reference | 2026-06-24 | Stateless client + RPC/HTTPS reference (`rook-cli` local-first TUI, `rook-server` Cloud Run microservices, `rook-server-cli` admin tool) — the category 1/3 precedent and the catalog cache-with-fallback pattern. |
| `monamaret/sight` | https://github.com/monamaret/sight | 2026-06-24 | Stateless client + RPC-over-SSH-auth reference (`sightd` server, `sight-cli` client) — note the server-side TUI removal in `specs/009-server-interface-prep`; source of the incremental-sync shape. |
| `charmbracelet/soft-serve` | https://github.com/charmbracelet/soft-serve | 2026-06-24 | External named precedent for category 4 — Kubernetes-hosted apps reachable only via an SSH-served TUI. |

## Appendix

None yet.
