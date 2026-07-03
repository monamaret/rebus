# Research: Generic web app + CLI scaffold (dual thin-client, optional MCP)

**Status:** Complete — all open questions resolved; ready for feature-item creation
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

The first release's chat app ([F-005](../features/005-private-chat-1to1.md))
was going to be the platform's first instance of the rook/sight
stateless-client pattern paired with *two* thin clients (TUI + web)
against one backend. The owner has decided to **drop the web (and mobile)
interface for chat specifically, for now**, and instead generalize that
dual-thin-client idea into its own reusable scaffold: a generic
**web app + CLI** template, built on the same stateless pattern, that
`skipper` can stamp out for future apps — proven out with one concrete
instance deployed in this release. This also introduces a new concern not
yet covered anywhere: **MCP (Model Context Protocol)** integration, both as
an optional server on the backend and as something the CLI interface
itself integrates with.

This directly bears on two of
[001-web-layer-templates.md](001-web-layer-templates.md)'s open questions:
whether Catalyst UI Kit's primary use is a new "application" template
distinct from content sites, and whether such a template is the web
counterpart the hybrid TUI layer's lazy-sync apps talk to. This scaffold
is that template.

- **Goal:** Define the scaffold's shape — what `skipper` generates, how
  the web app's static-vs-live configurability works, and how the optional
  MCP server/CLI integration fits — narrow enough to drive a feature item.
- **In scope:** The dual-client architecture pattern, static-vs-live web
  configurability, MCP server placement and CLI integration, and how this
  relates to the existing TUI shell ([../features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)) and chat ([../features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md)) feature items.
- **Out of scope:** Re-deciding chat's own architecture (already settled
  in [002](002-hybrid-tui-layer.md); only its web-client *deliverable* for
  v1 is being deferred, tracked in
  [005's feature spec](../features/005-private-chat-1to1.md)).
- **Success criteria:** A confirmed scaffold shape recorded in this doc's
  Decision Log, plus a feature item.

## Background & Context

- **Relevant prior work:** [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md)
  already established the stateless client + RPC/HTTPS pattern (categories
  1 and 3) and the WebAuthn web-auth decision — both apply directly here.
  [biblio](https://github.com/monamaret/biblio) already has a working MCP
  server pattern: an `/mcp` route mounted on the same Cloudflare Worker
  that serves the static site, returning Markdown/plain-text `content` as
  the primary response (so terminal-based MCP clients work), with a richer
  SVG/HTML MCP App UI resource layered on top as a bonus, never a
  requirement (biblio Principle IV, "Text-First, Progressive Enhancement").
  biblio also uses a single flat `mcp_enabled` boolean in `site.yaml` as
  the only MCP-related build-time toggle (biblio's Decision Log,
  "Single-Author Minimal-Dependency Bias" / "One Pipeline, One Source of
  Truth") — a precedent worth reusing for this scaffold's own
  static-vs-live and MCP-on/off configuration rather than inventing nested
  config.
- **Domain assumption:** This scaffold is infrastructure for *future* apps
  as much as it's a thing to use once — the v1 deliverable is both the
  scaffold (a `skipper` command that generates this shape) and one
  concrete deployed instance proving it works.

## Libraries & Stack

| Surface | Stack | Notes |
| --- | --- | --- |
| CLI | Go + Cobra, consistent with `skipper`'s own CLI conventions (AGENTS.md) | Thin client over the backend's stateless RPC/HTTPS API — no business logic duplicated client-side. |
| Web (static mode) | React + Vite, served from **Firebase Hosting** (GCP-managed CDN/static hosting) | No live backend calls at request time; same content-shaped idea as biblio's site types, but GCP-native instead of Cloudflare — see Decision Log. |
| Web (live mode) | React + Vite SPA shell on Firebase Hosting, with Hosting rewrites routing dynamic/API paths to **Cloud Run** (or a single Cloud Run service serving the whole app, if simplicity wins over edge-CDN benefits) | This is the "application" template gap [001](001-web-layer-templates.md) flagged — distinct from biblio's content-site types, and GCP-native per the Decision Log below. |
| Backend | Whatever [003-backend-application-deployment.md](003-backend-application-deployment.md)'s GitHub-repo build path produces, deployed on GKE | The actual stateless RPC/HTTPS service the CLI and web (live mode) talk to — reachable from Cloud Run over a private VPC path, never required to be public. |
| MCP (optional) | Same response-shape pattern as biblio's `/mcp` route — Markdown/plain-text-first tool responses, optional MCP App UI resource as a bonus | **Resolved (see Decision Log): lives on the backend (GKE), not the web layer's Cloud Run service** — no longer a "Worker," since the web layer is no longer on Cloudflare. |

**Selected:** GCP-native web hosting (Firebase Hosting + Cloud Run), not
Cloudflare Workers — see Decision Log.
**Open questions:** see below.

## Architecture Patterns

- **Web hosting is GCP-native (Firebase Hosting + Cloud Run), not
  Cloudflare Workers — this scaffold's web layer is a different category
  from biblio's standalone content sites.** biblio's Cloudflare choice was
  made for a problem with no GCP backend dependency at all (personal
  content sites). This scaffold is explicitly "a UI for backend
  applications" — a thin client wired to a GCP/GKE backend — and for that
  shape, staying GCP-native end to end avoids a second identity/access
  system (Cloudflare Access alongside GCP IAM + the WebAuthn registry), a
  second CLI/credential toolchain (Wrangler alongside `gcloud`), and a
  forced-public network path (a Cloudflare Worker can only reach the
  backend over the public internet; Cloud Run can reach it over a private
  VPC path instead, keeping the backend off the public internet entirely).
  Firebase Hosting was already named in this platform's founding
  description as an intended GCP managed service — this isn't a new
  dependency, just using one already in scope. See Decision Log.
  *(Deployment mechanism filled in — see
  [012-firebase-hosting-cloud-run-deployment.md → Decision Log](012-firebase-hosting-cloud-run-deployment.md#decision-log):
  Firebase CLI for Hosting, Cloud Run's own declarative YAML for live
  mode, persisted under `deployments/<app-name>/web/`, WIF-based CI/CD —
  mirroring `010`'s GKE/Kustomize pattern.)*
- **One backend, multiple thin surfaces, one set of operations.** The CLI,
  the web app (in live mode), and the optional MCP tool surface should all
  invoke the *same* underlying backend operations rather than each
  re-implementing logic — directly extending biblio's "One Pipeline, One
  Source of Truth" principle to a third surface (MCP) beyond the original
  two (site build + MCP data). *(Mechanism filled in — see
  [011-shared-operation-definitions-and-mcp-sdk.md → Decision Log](011-shared-operation-definitions-and-mcp-sdk.md#decision-log):
  the MCP server SDK is the official `modelcontextprotocol/go-sdk`; each
  shared operation is a plain Go function `func(ctx, Input) (Output,
  error)`, wrapped separately by an `mcp.AddTool` adapter and a Cobra
  command adapter, both calling the same function — no code generation;
  auth is enforced per-adapter, not centralized in the shared function.)*
- **Static vs. live is a per-instance config flag, not two different
  scaffolds.** Mirrors biblio's flat-boolean-config precedent
  (`mcp_enabled`) — e.g. a single `web_mode: static | live` field, not a
  nested config object — so `skipper` generates one scaffold shape with a
  config switch, not two divergent template trees.
- **MCP is opt-in per instance**, same flat-boolean precedent
  (`mcp_enabled`), independent of static/live — a static content app could
  still expose an MCP server over its content, and a live app could opt
  out of MCP entirely.
- **Auth reuses the WebAuthn decision from [002](002-hybrid-tui-layer.md)
  when live mode actually needs it.** Static mode needs no auth; live mode
  with any backend calls beyond public reads uses the same WebAuthn +
  owner-controlled registry pattern already decided for chat, rather than
  inventing a second web-auth mechanism.
- **Tradeoffs:** Supporting four real axes of variation (CLI always-on;
  web static/live; MCP on/off) is more scaffold-generation complexity than
  a single fixed shape — mitigated by keeping MCP and live-mode strictly
  optional/off-by-default, so the simplest instantiation (CLI + static
  web, no MCP) stays as simple as a biblio-style content site plus a thin
  CLI wrapper.

## Existing Codebase Evaluation

No `skipper` scaffolding code exists yet. biblio remains the closest
existing reference for both the static-web-template mechanics and the MCP
server pattern; this scaffold extends biblio's pattern with a paired CLI
and the static/live + MCP configurability biblio itself doesn't have.

## Security & Compliance

- Live-mode web auth reuses the WebAuthn/owner-registry decision from
  [002 → Decision Log](002-hybrid-tui-layer.md#decision-log) — no new
  mechanism.
- **MCP surface auth posture: resolved — defaults to access-controlled,
  not public/no-auth** (see Decision Log), unlike biblio's own MCP
  endpoint, which is intentionally public/no-auth for low-sensitivity,
  read-oriented content. This scaffold's MCP surface instead reuses the
  same WebAuthn/key-registry auth as live-mode web, since it may wrap
  arbitrary, non-public backend operations rather than read-only content.

## Performance, Reliability & Operations

Not yet applicable — no implementation exists.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Four-axis configurability (CLI/web static/web live/MCP) becomes a maintenance burden across every future app built from this scaffold | Medium | Medium | Keep MCP and live-mode strictly opt-in/off-by-default; the default instantiation stays as simple as a CLI + static site | mona |
| MCP surface inherits biblio's public-no-auth posture by default but actually wraps non-public backend operations | Low–Medium | Medium–High | Decide MCP auth posture explicitly per instance, not by blindly copying biblio's content-site default | mona |
| CLI and MCP tool definitions drift apart if not generated from/kept in sync with one shared source | Medium | Medium | **Resolved by [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md):** each operation is one plain Go function, wrapped separately by an MCP adapter and a CLI adapter that both call it — no drift possible since both adapters share the same function | mona |

## Open Questions

All five questions originally listed here are resolved — see the Decision
Log. Remaining work is implementation-scoping (feature item creation), not
further research.

## Decision Log

- **2026-06-24** — **This scaffold's web layer is hosted on GCP-native
  services — Firebase Hosting for the static shell, Cloud Run for any
  live/dynamic component (via Hosting rewrites, or as the sole service if
  simplicity outweighs the edge-CDN benefit) — not Cloudflare Workers.**
  — Rationale: this scaffold's web app is explicitly "a UI for backend
  applications," a thin client wired to a GCP/GKE backend working with
  multiple GCP and Kubernetes services — a fundamentally different shape
  from biblio's standalone, backend-independent content sites that
  justified Cloudflare. Staying GCP-native: (1) keeps one identity/access
  model (GCP IAM + the WebAuthn registry) instead of adding Cloudflare
  Access as a second one; (2) keeps one CLI/credential toolchain (`gcloud`)
  instead of adding Wrangler as a second; (3) allows the web layer to
  reach the backend over a private VPC path (Cloud Run → GKE) instead of
  forcing the backend to be publicly reachable for a Cloudflare Worker to
  call it; (4) uses Firebase Hosting, already named in this platform's
  founding description as an intended GCP managed service. biblio's
  Cloudflare pattern remains the right choice for the *separate* category
  of standalone, backend-independent content sites ([001-web-layer-templates.md](001-web-layer-templates.md))
  — this decision does not reverse that, it scopes GCP-native hosting to
  this scaffold's backend-application-UI category specifically. — Decided
  by: mona.
- **2026-06-24** — **The optional MCP server lives on the backend (GKE),
  alongside the stateless RPC API — not on the web layer's Cloud Run
  service.** — Rationale: the backend is where the actual operations/logic
  live; routing MCP tool calls through Cloud Run would add a redundant
  proxy hop with no benefit, and conflicts with the "one backend, multiple
  thin surfaces, one set of operations" pattern already established above.
  — Decided by: mona.
- **2026-06-24** — **"MCP App integration with the CLI tool interface"
  means CLI commands and MCP tools are generated from one shared
  operation-definition set — not that the CLI itself renders or triggers
  MCP App UI resources.** — Rationale: the CLI is a terminal tool with no
  native HTML/SVG rendering; having it render MCP App UI resources is
  speculative complexity with no demonstrated need (Principle I). Shared
  operation definitions avoid duplicating logic between the CLI and MCP
  surfaces, which is the actual problem worth solving here. — Decided by:
  mona.
- **2026-06-24** — **The MCP surface defaults to access-controlled, not
  public/no-auth, reusing the same WebAuthn/key-registry auth as live-mode
  web.** — Rationale: unlike biblio's MCP endpoint (intentionally public,
  low-sensitivity, read-oriented content), this scaffold may wrap
  arbitrary, non-public backend operations — defaulting to public/no-auth
  would be an unsafe default for the general case, even though any
  specific instance could still choose to expose a public, read-only MCP
  surface deliberately. — Decided by: mona.
- **2026-06-24** — **The v1 example app instantiated from this scaffold
  defaults to static web mode + MCP on; live web mode is deferred.** —
  Rationale: the CLI is the primary live/interactive surface and needs no
  new auth plumbing to prove; static web needs no auth at all; MCP on
  proves out the actually-novel capability this release wants to
  demonstrate. Live web mode (and the WebAuthn auth flow it requires) is
  deferred to a later proof — e.g. when chat ([../features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md))
  eventually adopts this scaffold for its own deferred web client. —
  Decided by: mona.
- **2026-06-24** — **This scaffold's app registers as an app-catalog entry
  for discoverability, but the TUI shell launches it by shelling out to
  the CLI binary rather than embedding a separate Bubble Tea view.** —
  Rationale: the CLI already covers the interactive-terminal use case;
  building a third UI surface (CLI, web, *and* a TUI-shell-native view) for
  one app isn't justified yet. Catalog registration keeps this app
  consistent with every other stateless app's discoverability story
  without that extra surface. — Decided by: mona.
- **2026-06-24 (concretizes the v1 example app)** — **The v1 instance of
  this scaffold is a platform deployment/admin dashboard**, surfacing both
  app-catalog data and live GKE health (pod status, restarts, resource
  usage) — raised while refining
  [003-off-the-shelf-image-deployment.md](../features/003-off-the-shelf-image-deployment.md),
  which needed a home for "view the deployment in a web UI with admin
  stats." It uses the already-decided v1 defaults unchanged (static web +
  MCP on) — no reversal to live mode needed. — Decided by: mona.
- **2026-06-24** — **MCP tool responses for this dashboard include a
  plain-text/terminal-renderable visualization tier (e.g. ASCII bar charts,
  sparklines) that the TUI can render directly — distinct from, and not a
  reversal of, the earlier "CLI does not render MCP App UI" decision.**
  — Rationale: this directly extends biblio's already-established
  two-tier MCP response pattern (Markdown/plain-text primary content,
  richer SVG/HTML MCP App UI resource as an optional bonus) — simple
  ASCII/Unicode visualizations are part of the *plain-text* tier, not the
  rich HTML/SVG tier, so the TUI rendering them is consistent with "CLI/
  MCP integration = shared operation definitions only": the CLI/TUI
  surfaces whatever tier of content those shared operations return, which
  may include simple structured visualizations, but still never renders
  the rich MCP App UI resource layer (that stays web-only). Lets the same
  MCP tool call serve both the static web dashboard and a TUI-plugged-in
  view with no duplicated visualization logic. — Decided by: mona.

> **Format note (2026-07-03):** this doc predated the current
> `research-template.md`; its single `## References` list has been split into
> `## Internal References` + `## Web Citations` to match the template.

## Internal References

- [001-web-layer-templates.md](001-web-layer-templates.md) — open
  questions this doc narrows (Catalyst "application" template; web
  counterpart to TUI lazy-sync apps)
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — stateless
  client + RPC pattern and WebAuthn auth decision this scaffold reuses
- [003-backend-application-deployment.md](003-backend-application-deployment.md) — deployment path for this scaffold's backend
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — Kustomize manifest mechanism this scaffold's backend deployment uses
- [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) — the concrete MCP SDK and shared-operation mechanism this doc's "one operation, multiple surfaces" decision implements
- [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md) — the concrete Firebase Hosting/Cloud Run deployment mechanism this doc's GCP-native web-hosting decision implements
- [F-005 (private chat)](../features/005-private-chat-1to1.md) — chat's web client deferral, the immediate trigger for this doc

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `monamaret/biblio` | https://github.com/monamaret/biblio | 2026-06-24 | The MCP server pattern (Markdown/plain-text-first `/mcp` responses, optional MCP App UI resource) and flat-boolean config precedent (`mcp_enabled`) this scaffold extends. |

## Appendix

None yet.
