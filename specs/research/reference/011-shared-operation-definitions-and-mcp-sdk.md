# Research: Shared CLI/MCP operation definitions and the MCP server SDK

**Status:** Complete — SDK choice and operation-sharing pattern resolved directly; one related follow-up (Firebase/Cloud Run deployment mechanics) flagged for a future doc; ready to update specs/research/005-generic-app-scaffold.md and specs/features/006-deployment-admin-dashboard.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[005-generic-app-scaffold.md](005-generic-app-scaffold.md) decided that
the generic scaffold's CLI and MCP tool surfaces "are generated from/call
into the same backend operations — no duplicated logic between the two
surfaces" — but never specified *how*: no Go MCP server library was
chosen, and no concrete shape for "one operation, two surfaces" was
designed. [006-deployment-admin-dashboard.md](../features/006-deployment-admin-dashboard.md)
(the scaffold's v1 instance) still carries this forward as an unspecified
"shared operation-definition mechanism" Scope bullet. This doc fills in
both gaps: the MCP server SDK, and the concrete pattern for sharing one
operation between a Cobra CLI command and an MCP tool.

- **Goal:** Select a Go MCP server SDK and define the concrete
  shared-operation pattern — narrow enough to update `005` and `006` with
  a real mechanism instead of a placeholder phrase.
- **In scope:** MCP server SDK comparison; the function-signature shape
  an "operation" takes; how that one definition is wired into both
  `mcp.AddTool` (or equivalent) and a Cobra command.
- **Out of scope:** Re-opening `005`'s already-settled decisions (MCP on
  the backend not Cloud Run, access-controlled by default, plain-text/
  terminal-renderable response tier) — not reconsidered here. Firebase
  Hosting + Cloud Run's own deployment mechanics (analogous to what `010`
  did for GKE/Kustomize) are a related but separate gap, flagged as a
  follow-up in Open Questions rather than covered in this doc.
- **Success criteria:** A confirmed SDK choice and operation-sharing
  pattern, recorded in this doc's Decision Log, ready to update `005`/`006`.
- **Non-goals:** Designing the dashboard's actual operations (e.g. the
  exact `dashboard stats` tool/command) — that's `006`'s own
  implementation detail once this mechanism exists.

## Background & Context

- **Relevant prior work:**
  - [005 → Decision Log](005-generic-app-scaffold.md#decision-log) — MCP lives on the backend; CLI and MCP tools share one operation-definition set; MCP defaults to access-controlled.
  - [006-deployment-admin-dashboard.md → Scope](../features/006-deployment-admin-dashboard.md) — "Shared operation-definition mechanism so CLI commands and MCP tools are generated from/call into the same backend operations" — the unspecified mechanism this doc defines.
  - [.specify/memory/constitution.md Principle III](../../.specify/memory/constitution.md) — idiomatic Go conventions (standard project layout, wrapped errors, structured logging) apply to whatever shape is chosen here.
- **Domain assumption:** MCP itself (the protocol) is unaffected by this
  doc — only the Go-side SDK and the operation-sharing pattern are in
  scope.
- **Stakeholders:** mona (sole owner/operator and sole developer of every
  app built from this scaffold).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **`modelcontextprotocol/go-sdk`** (official) | MCP server/client implementation | MIT/Apache-2.0-style (official MCP org license) | Mature — v1.6.1, 26 releases, no stability disclaimers | Maintained by Anthropic **in collaboration with Google** | High | Confirmed via the SDK's own repo (see Web Citations): production-presented, no "not yet stable" caveats. The Google-collaboration angle is a soft but real alignment signal for a GCP-centric platform. |
| `mark3labs/mcp-go` | MCP server/client implementation | MIT | Mature — imported by 400+ packages across 200+ modules | Active community maintenance | Medium-High | A real, battle-tested alternative — but a third-party implementation that risks diverging from spec updates now that an official SDK exists, where the official one doesn't carry that risk. |
| `mcp-golang` | MCP server/client implementation | MIT | Less prominent / smaller adoption than the above two | Unclear, smaller community | Rejected | No adoption signal strong enough to prefer over either the official SDK or `mark3labs/mcp-go` — no reason to choose the least-adopted option among three credible candidates. |

**Selected:** `modelcontextprotocol/go-sdk` (official). Its tool-handler
shape — a plain Go function `func(ctx, req, Input) (*mcp.CallToolResult,
Output, error)` with typed `Input`/`Output` structs (see Web Citations)
— is also the concrete basis for the shared-operation pattern below.
**Rejected (with reason):** `mark3labs/mcp-go` — a credible, more
battle-tested alternative, but the official SDK's existence and
production-ready status removes the main reason to prefer a third-party
implementation; `mcp-golang` — weaker adoption signal than either
alternative, no offsetting advantage.
**Open questions:** none for this section — resolved from the SDK
comparison directly, not an owner preference call.

## Architecture Patterns

- **One operation = one plain Go function with typed `Input`/`Output`
  structs — no MCP-specific or Cobra-specific types in its own
  signature.** Directly reusing the official SDK's own handler shape
  (`func(ctx context.Context, req *mcp.CallToolRequest, input Input)
  (*mcp.CallToolResult, Output, error)`, see Web Citations) as the
  canonical operation signature, *minus* the MCP-specific `req`/result
  wrapper for the operation's actual business logic — i.e. the operation
  itself is `func(ctx, Input) (Output, error)`, and two thin adapters
  wrap it:
  - **MCP adapter**: registers `mcp.AddTool(server, &mcp.Tool{Name: ...,
    Description: ...}, func(ctx, req, input Input) (*mcp.CallToolResult,
    Output, error) { out, err := operation(ctx, input); return
    mcp.NewToolResultText(...), out, err })` — a one-line wrapper per
    operation.
  - **CLI adapter**: a Cobra command whose `RunE` parses flags/args into
    the same `Input` struct, calls `operation(ctx, input)` directly, and
    renders `Output` to stdout (the plain-text/terminal-renderable tier
    already decided in `005`).
  - Both adapters call the *same* `operation` function — this is what
    "no duplicated logic between the two surfaces" concretely means.
- **The dashboard's `dashboard stats` operation is the first concrete
  instance** of this pattern: `Input` is empty (or carries optional
  filter flags), `Output` carries the catalog/GKE-health data plus the
  ASCII-chart-ready fields already decided in `005`/`006`.
- **No code generation step** — operations are hand-written Go functions
  registered explicitly in both adapters, not generated from an external
  schema (e.g. OpenAPI/protobuf). This matches Principle I: a code-gen
  pipeline is real infrastructure unjustified for the number of
  operations this scaffold actually needs per app.

**Tradeoffs:** Hand-registering each operation in two adapters (MCP +
CLI) is two lines of boilerplate per operation, in exchange for avoiding
a code-generation toolchain that would be disproportionate machinery for
a single-owner platform's actual operation count per app (Principle I).
**Integration points:** the plain-text/terminal-renderable response tier
already decided in `005` is exactly what the CLI adapter renders to
stdout — no new rendering logic, the same `Output` struct serves both
the MCP tool's text content and the CLI's terminal output.
**Data model implications:** none beyond the `Input`/`Output` struct
shapes themselves, which are per-operation and app-specific (e.g. the
dashboard's stats `Output`), not a shared schema across apps.

## Existing Codebase Evaluation

No `skipper` scaffold code exists yet. The official SDK's own
`quick_start` example is the direct, literal precedent for the operation/
adapter shape above — not a `skipper`-internal pattern being reused, but
an external SDK's own idiom adopted as the project's convention.

## Security & Compliance

- The MCP adapter's access-control enforcement (already decided in `005`
  — access-controlled by default, reusing WebAuthn/the owner registry)
  wraps the MCP-facing adapter specifically, not the shared `operation`
  function itself — the CLI adapter has its own auth path (the
  SSH-key-based session already used platform-wide), so access control
  is enforced per-adapter, not centrally in the shared operation. This is
  a deliberate consequence of keeping the shared function free of
  surface-specific concerns (Principle III's standard-layout/clean-
  separation spirit) — each adapter is responsible for its own auth
  before calling the shared operation.

## Performance, Reliability & Operations

Not yet applicable beyond what's already covered in `005`/`006` — this
doc is about code structure, not a new runtime characteristic.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| The official SDK, despite being production-presented, is younger than `mark3labs/mcp-go` and could see faster breaking changes as the protocol itself evolves | Low–Medium | Medium | Pin the SDK version explicitly (Go modules already do this); revisit if a real breaking-change pain point shows up in practice | mona |
| Forgetting to enforce auth in one adapter (e.g. a new operation's CLI command skips the session check) since auth isn't centralized in the shared operation function | Low–Medium | Medium–High | Document the per-adapter-auth convention clearly (this doc + a code comment at the adapter layer) so it's an explicit, known pattern, not an easy-to-forget implicit assumption | mona |

## Open Questions

None for the scope of this doc — both the SDK choice and the operation-
sharing pattern are resolved directly above.

- [x] ~~Firebase Hosting + Cloud Run deployment mechanics~~ — **Resolved
  by [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md)**:
  Firebase CLI for Hosting, Cloud Run's own declarative YAML
  (`gcloud run services replace`) for live mode, persisted under
  `deployments/<app-name>/web/`, WIF-based CI/CD.

## Decision Log

- **2026-06-24** — **`modelcontextprotocol/go-sdk` (the official MCP Go
  SDK) is the chosen MCP server library — not `mark3labs/mcp-go` or
  `mcp-golang`.** — Rationale: production-presented (v1.6.1, 26 releases,
  no stability disclaimers), maintained by Anthropic in collaboration
  with Google — a soft alignment signal for this GCP-centric platform;
  removes the main reason (spec-divergence risk) to prefer a third-party
  implementation. — Decided by: mona.
- **2026-06-24** — **Each shared operation is a plain Go function
  `func(ctx, Input) (Output, error)`, with no MCP- or Cobra-specific
  types in its own signature; an MCP adapter and a CLI adapter each wrap
  it separately, both calling the same function.** No code generation —
  operations are hand-registered in both adapters. — Rationale: directly
  reuses the official SDK's own tool-handler shape as the canonical
  signature; avoids a code-gen toolchain disproportionate to this
  scaffold's actual per-app operation count (Principle I). — Decided by:
  mona.
- **2026-06-24** — **Auth is enforced per-adapter, not centralized in the
  shared operation function** — the MCP adapter enforces the
  access-controlled-by-default WebAuthn check (per `005`); the CLI
  adapter relies on the platform's existing SSH-key-based session. —
  Rationale: keeps the shared operation free of surface-specific
  concerns; the tradeoff (each new operation's adapters must remember to
  enforce auth) is mitigated by documenting the convention explicitly
  rather than centralizing in a way that would couple the operation to
  one specific auth mechanism. — Decided by: mona.

## Internal References

- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — the scaffold decision this doc fills in the unspecified mechanism for
- [006-deployment-admin-dashboard.md (feature)](../features/006-deployment-admin-dashboard.md) — the feature item this research will refine
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — the analogous prior gap-filling doc this one mirrors in structure, and the source of the Firebase/Cloud Run follow-up flagged above
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and Principle III (idiomatic Go), both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `modelcontextprotocol/go-sdk` — Quick Start | https://go.sdk.modelcontextprotocol.io/quick_start/ | 2026-06-24 | Source of the exact tool-handler signature and `mcp.AddTool` registration call this doc's shared-operation pattern is built from. |
| `modelcontextprotocol/go-sdk` — GitHub repo | https://github.com/modelcontextprotocol/go-sdk | 2026-06-24 | Confirms current release status (v1.6.1, 26 releases), maintainer (Anthropic, in collaboration with Google), and the absence of any stability disclaimer — basis for selecting it over the community alternatives. |
| Web search — "Go MCP server SDK library model context protocol golang" | (search query, no single URL) | 2026-06-24 | Identified the three credible Go MCP SDK candidates compared in Libraries & Stack (`modelcontextprotocol/go-sdk`, `mark3labs/mcp-go`, `mcp-golang`) and `mark3labs/mcp-go`'s adoption figure (400+ importing packages). |

## Appendix

None yet.
