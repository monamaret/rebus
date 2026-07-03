# Research: Public client-package contract — the cross-repo Go API `kingfish`/`bateau` import

**Status:** Complete — the shared shape both backend client packages follow is
recorded; per-backend method sets are finalized in each backend's feature item
**Date:** 2026-07-03
**Owner:** mona

## Problem Statement

`pocket` (stash backend) and `rebus` (chat backend) each "publish a public
(not `internal/`) Go client package" that other repos import at compile time:
`kingfish`'s embedded stash/chat views import `pocket`'s and `rebus`'s
packages, and `bateau` imports `rebus`'s (`R-006`, `R-013`, `R-018`, `R-023`).
Eight docs reference "the client package," but none specify its **API
surface** — the method set, wire types, error contract, or versioning. This is
a cross-repo *compile-time* dependency: the Go toolchain forbids importing an
`internal/` package across module boundaries (Web Citations), which is exactly
why these must be **public** packages — and public packages are an API
contract that, once other repos import them, is expensive to change silently.
This doc defines the **shared shape** both packages follow so `pocket` and
`rebus` don't diverge, and so the importing repos can be specified against a
known surface.

- **Goal:** Define the common structure, error contract, sync-method shape,
  and versioning discipline every `skipper`-platform backend's public client
  package follows — enough to specify both the backends and their importers.
- **In scope:** Package layout and import path; the `Client` surface shape;
  wire-type placement; the exported error contract; the incremental-sync
  method shape; auth injection; SemVer/compatibility rules.
- **Out of scope:** The exact, complete method list for each specific backend
  (that belongs in each backend's own feature item — `pocket` → `F-004`,
  `rebus` → `F-005` — since the operations differ); the RPC wire format/
  framing itself (an implementation detail below the client's Go surface).
- **Success criteria:** A confirmed shared client-package contract recorded in
  this doc's Decision Log, referenced by `F-004`, `F-005`, and the
  `kingfish`/`bateau` begin-state specs.
- **Non-goals:** A single shared client *library* both backends depend on —
  each backend owns and publishes its own package; this doc standardizes their
  *shape*, not a common dependency (Principle I — no premature shared module).

## Background & Context

- **Relevant prior work:**
  - `R-002` → the stateless-client + RPC/HTTPS pattern (rook/sight); SSH-key-
    signed challenge auth, no long-lived credential leaves the client.
  - `R-006` ([006-private-chat-application.md](006-private-chat-application.md))
    → `rebus`'s "one shared, public Go package for wire types/RPC calls,
    imported by both the owner's TUI view and the guest's standalone client,"
    adapted from `sight`'s import-boundary pattern; clients never touch
    Firestore directly (backend-RPC-mediated only).
  - `R-018` ([018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md))
    → `pocket` publishes a public (not `internal/`) Go client package that
    `kingfish`'s embedded stash view imports.
  - `sight`'s `NoteService.SyncNotes(ctx, customerID, since time.Time)` — the
    concrete incremental-sync method shape both backends reuse (`R-002`).
- **Domain assumption:** The importers (`kingfish`, `bateau`) are the *only*
  consumers today, all owner-controlled — but they are separate repos/modules,
  so the package is a real cross-repo contract regardless of single ownership.
- **Stakeholders:** mona (owns every repo on both sides of the boundary).

## Libraries & Stack

Not a new-library evaluation — this standardizes a Go package *shape* using
the stdlib + the transport each backend already uses (RPC/HTTPS per `R-002`).
The one external fact that constrains the design is the Go toolchain's
`internal/` cross-module rule (Web Citations): a package imported by another
repository **must not** live under `internal/`, so these are ordinary
public packages, versioned by Go modules.

**Selected shape:** each backend publishes one public package (e.g.
`github.com/monamaret/pocket/client`) containing both the `Client` and the
wire types, versioned by the backend module's own SemVer tags.
**Rejected:** a separate shared `github.com/monamaret/<x>-clientkit` module both
backends depend on — premature shared dependency (Principle I) with only two
backends that don't actually share operations; the *convention* is shared, the
*code* is not.

## Architecture Patterns

- **Package layout — one public package per backend, wire types co-located.**
  `github.com/monamaret/pocket/client` and `github.com/monamaret/rebus/client`
  (name illustrative; each backend's feature item confirms its own). The
  package exports the `Client`, its constructor, the request/response wire
  types, and the error sentinels. Not under `internal/` (cross-repo import
  would fail to compile — Web Citations). Following `sight`'s bounded
  wire-type-package boundary, but **public** rather than `internal/` because
  the consumers are in other modules.
- **`Client` surface — a concrete constructor + one method per backend op.**

  ```go
  package client

  type Client struct { /* unexported: http client, base URL, auth */ }

  func New(cfg Config) (*Client, error)   // cfg carries base URL + an Authenticator

  // one method per backend RPC, e.g. (pocket):
  //   func (c *Client) List(ctx context.Context, in ListInput) (ListOutput, error)
  //   func (c *Client) Get(ctx, GetInput) (GetOutput, error)
  //   func (c *Client) Put(ctx, PutInput) (PutOutput, error)
  //   func (c *Client) Sync(ctx, since time.Time) (SyncOutput, error)
  ```

  Every method takes `context.Context` first and returns
  `(TypedOutput, error)`. The concrete method list per backend is defined in
  that backend's feature item, not here — this doc fixes the *shape*
  (ctx-first, typed in/out, error last), not the roster.
- **Auth is injected, not baked in.** `Config` carries an `Authenticator`
  interface the caller supplies, so the same package serves `kingfish`
  (owner's SSH key) and `bateau` (guest's SSH key, resolved SSH-agent-first
  per `R-014`) without the package knowing which. Consistent with `R-002`'s
  "no long-lived credential leaves the client" — the package signs challenges
  via the injected `Authenticator`, it never stores a key.
- **Incremental sync is a first-class method, `sight`-shaped.** A
  `Sync(ctx, since time.Time)` (or equivalent) returning everything modified
  at/after `since`, mirroring `sight`'s `SyncNotes(ctx, customerID, since)`.
  `kingfish`'s embedded views and `bateau` both drive their `updatedAt`-
  incremental refresh through this method — it is part of the public contract,
  not an internal detail, because the importers depend on it directly.
- **Error contract — exported sentinels, not string-matching.** The package
  exports typed sentinel errors (at minimum `ErrNotFound`, `ErrUnauthorized`,
  `ErrConflict`) that importers match with `errors.Is`, so `kingfish`/`bateau`
  branch on outcome without parsing message strings. Transport/unknown errors
  wrap the underlying error (`fmt.Errorf("...: %w", err)`) so callers can still
  inspect them.
- **Versioning — Go module SemVer; additive by default.** Each backend module
  tags releases; importers pin a version in `go.mod`. Backward-compatible
  changes (new methods, new optional fields on wire structs) are minor/patch;
  a breaking surface change is a new major version (new module path suffix per
  Go's rules), never a silent break — because the importers are separate
  repos, a silent break would surface only at *their* next build.
- **Tradeoffs:** Duplicating the same package *shape* in two repos (rather than
  a shared module) means the convention is enforced by this doc + review, not
  by a compiler — accepted under Principle I, since two backends with disjoint
  operation sets don't justify a shared module and the coupling that creates.
- **Integration points:** `pocket`↔`kingfish` stash view; `rebus`↔`kingfish`
  chat view; `rebus`↔`bateau`. All three are the same contract shape.
- **Data model implications:** wire types are the backends' *public* API types,
  deliberately distinct from their internal Firestore/GCS storage models
  (clients never see storage shapes — `R-006`'s "clients never access
  Firestore directly").

## Existing Codebase Evaluation

No `skipper`-platform client packages exist yet — all five real component
repos are zero-code (`R-023`). `sight`'s wire-type-package + `SyncNotes`
boundary and `rook-reference`'s local-first client are the reference shapes
(cited in `R-002`); neither is vendored or forked.

## Security & Compliance

- **No credential storage in the package.** Auth is via the injected
  `Authenticator` (SSH-key-signed challenge, `R-002`/`R-014`); the package
  holds no key and no long-lived token — session material is the caller's
  concern, in-memory per `R-002`.
- **Least-surface public API.** Only the operations importers actually need
  are exported; nothing exposes the backend's storage model or admin
  internals. `rebus`'s admin operations (`R-013`) are gated at the backend's
  adapter layer and are *not* part of `bateau`'s consumable surface even
  though `bateau` imports the same package — the package can expose admin
  methods that the backend authorizes only for the owner's key.

## Performance, Reliability & Operations

- Thin client over the backend's RPC/HTTPS — performance is the backend's
  concern; the package adds only marshaling + auth-header attachment.
- The `Sync(since)` method's incremental contract (return only what changed)
  is what keeps `kingfish`/`bateau` refresh cheap; a full-list fallback exists
  for first sync (empty `since`).

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| `pocket` and `rebus` client packages drift into different shapes (method/err conventions) | Medium | Medium | This doc is the shared convention both feature items (`F-004`/`F-005`) build against; enforced by review, not compiler | mona |
| A breaking change to a public wire type silently breaks `kingfish`/`bateau` at their next build | Medium | Medium | SemVer discipline: breaking = new major (module path suffix); importers pin versions in `go.mod` | mona |
| A backend accidentally publishes client types under `internal/`, making cross-repo import fail to compile | Low | Medium | Explicit: the package is public (not `internal/`) — the whole point of the cross-repo boundary (Web Citations) | mona |

## Open Questions

- [ ] Final package **name/path** per backend (`.../client` vs `.../<name>`) —
  each backend's feature item confirms its own; convention only, not a
  research blocker. — owner: mona
- [ ] Whether the `Authenticator` interface itself should live in a tiny shared
  module (the one thing genuinely common to both) or be duplicated per package
  — revisit only if a third backend appears (Principle I). — owner: mona

## Decision Log

- **2026-07-03** — **Each backend publishes one public (not `internal/`) Go
  package containing its `Client`, wire types, and error sentinels, versioned
  by that backend module's own SemVer; no shared client module.** — Rationale:
  the Go toolchain forbids cross-repo `internal/` imports (Web Citations), so
  the consumer-facing package must be public; a shared module isn't justified
  for two backends with disjoint operation sets (Principle I) — the *shape* is
  standardized here, the *code* stays per-backend. — Decided by: mona.
- **2026-07-03** — **Standard surface: `New(cfg) (*Client, error)`; every op
  is `func(ctx, TypedInput) (TypedOutput, error)`, ctx-first; auth via an
  injected `Authenticator` in `cfg`; a `sight`-shaped `Sync(ctx, since)`
  incremental method is part of the public contract.** — Rationale: matches
  the rook/sight stateless-client pattern (`R-002`) and the shared-operation
  `func(ctx, Input) (Output, error)` shape already chosen for MCP/CLI adapters
  in `R-011`; injected auth lets one package serve both `kingfish` and
  `bateau` without embedding key handling. — Decided by: mona.
- **2026-07-03** — **Error contract is exported typed sentinels
  (`ErrNotFound`/`ErrUnauthorized`/`ErrConflict`, matched via `errors.Is`),
  with transport errors wrapped (`%w`); importers never string-match.** —
  Rationale: a stable, machine-checkable error surface is part of a public API
  contract across repos; string-matching would silently break on message
  changes. — Decided by: mona.
- **2026-07-03** — **Breaking changes to the public surface require a new
  major module version; additive changes are minor/patch.** — Rationale:
  importers are separate repos that pin versions — SemVer is the only thing
  preventing a silent cross-repo break at the importer's next build. — Decided
  by: mona.

## Internal References

- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — stateless-client RPC
  pattern, SSH-key auth, no-credential-leaves-client, `sight` sync shape
- [006-private-chat-application.md](006-private-chat-application.md) — `rebus`'s shared public package imported by the TUI view and `bateau`; clients never touch Firestore
- [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) — the `func(ctx, Input) (Output, error)` op shape this contract mirrors
- [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) — `rebus` admin ops gated at the adapter, not exposed to `bateau`
- [014-guest-chat-client-scope-and-auth-ux.md](014-guest-chat-client-scope-and-auth-ux.md) — `bateau`'s SSH-agent-first auth, injected via the `Authenticator`
- [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) — `pocket`'s public client package imported by `kingfish`
- [023-platform-repo-inventory.md](023-platform-repo-inventory.md) — the repos on both sides of this boundary
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (no premature shared module), Principle VI (least public surface, no stored credentials)

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Go command — Internal Directories | https://pkg.go.dev/cmd/go#hdr-Internal_Directories | 2026-07-03 | The Go toolchain forbids importing an `internal/` package from outside its parent module — so a package imported across repos (as `kingfish`/`bateau` import `pocket`/`rebus`) must be public, which is what makes it a real, stability-bound API contract. |

## Appendix

None yet.
