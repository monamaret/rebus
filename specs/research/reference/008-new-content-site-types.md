# Research: Concrete design of the three new content-site types

**Status:** Complete — all open questions resolved; ready for feature-item creation (phased: changelog, then api-docs, then broadcast)
**Date:** 2026-06-24
**Owner:** mona

**Scope note (added 2026-06-24):** [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md)
resolved the command/lifecycle mechanism around the three types designed
here: each ships via `skipper site new --type=changelog|api-docs|broadcast
<repo-name>`, which wraps `biblio-cli new` directly (a new, separate repo
per constitution Principle VIII) — `biblio-cli` itself then owns every
lifecycle action afterward (authoring, `update`, `build`, `deploy` via
`wrangler deploy`), not `skipper`. **This is gated on `biblio-cli` adding
support for these three new `--type` values** — tracked as a real
dependency in `017`'s Risks, not yet resolved in `biblio-cli` itself.

## Problem Statement

[001-web-layer-templates.md](001-web-layer-templates.md) settled the
*policy* questions for the standalone-content-site category: Transmit,
Commit, and Protocol each become a **new sibling site type** (not
upgrades to `log`/`docs`); all stay on **plain Markdown + frontmatter**
(no MDX); Catalyst stays out of this category (shadcn/ui-on-Radix only);
TypeScript-only. What 001 did not do — by design, it explicitly named this
as out of scope — is define each new type's actual shape: frontmatter
schema, route structure, and how each example's signature features
(episode embeds, RSS, per-topic nav + search + syntax highlighting)
translate onto biblio's real stack (React Router v7 native `prerender`,
Cloudflare Workers, `react-markdown`/`remark-gfm`) instead of the
examples' Next.js/MDX implementations.

- **Goal:** A concrete frontmatter schema, route/file structure, and
  build-pipeline shape for each of the three new site types — `broadcast`
  (Transmit-inspired), `changelog` (Commit-inspired), and `api-docs`
  (Protocol-inspired) — narrow enough to drive feature items.
- **In scope:** Frontmatter fields per type; RSS feed generation on Vite/
  React Router v7/Cloudflare Workers (no Next.js route handlers); syntax
  highlighting and search without MDX; per-topic sidebar navigation.
- **Out of scope:** Re-opening 001's policy decisions (new types vs.
  upgrades, no-MDX, no-Catalyst, TypeScript-only) — all settled, not
  reconsidered here.
- **Success criteria:** Each type's schema/structure confirmed in this
  doc's Decision Log, ready to become (or extend) feature items.
- **Non-goals:** Final visual/CSS design — that's implementation detail
  once a feature item exists, same non-goal 001 already established.

## Background & Context

- **Relevant prior work:** [001 → Decision Log](001-web-layer-templates.md#decision-log)
  — the four settled policy decisions this doc builds on, not reopens.
  biblio's existing `docs` type frontmatter (`title`/`order`/`section`/
  `summary`, no `tags`/`project` — confirmed in biblio's own Decision Log,
  cited in this platform's earlier research) is the direct precedent for
  `api-docs`'s per-topic nav fields.
- **Domain assumptions:** All three new types are prerendered static
  sites (React Router v7 native `prerender`), same as biblio's existing
  `profile`/`log`/`docs` — none of the three need a live backend, since
  this category is explicitly the *standalone, backend-independent*
  content-site category (constitution Principle VIII), distinct from the
  GCP-native backend-application-UI category.
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| React Router v7 **resource routes** (no default export, only a `loader` returning a `Response`) | RSS/XML feed generation for `changelog`, replacing Next.js's `feed.xml/route.ts` | MIT (React Router) | Stable, documented core feature | Actively maintained (Remix/React Router team) | High — framework-native, no extra dependency beyond what's already used | Confirmed via React Router's own docs (see Web Citations): "a route becomes a resource route by convention when its module exports a loader... but does not export a default component" |
| `feed` (npm) | XML feed serialization | MIT | Mature, widely used (also Commit's own choice) | Actively maintained | High — framework-agnostic; same package Commit uses, just called from a resource-route loader instead of a Next.js route handler | No code changes needed to the library itself, only to where it's invoked |
| `rehype-pretty-code` (Shiki-powered) or `react-shiki` | Syntax highlighting for `api-docs` code blocks, without MDX | MIT | Mature, actively maintained | Actively maintained | High — both integrate directly with `react-markdown`'s existing rehype-plugin pipeline (`remarkParse` → `remarkRehype` → a highlighting rehype plugin → `rehypeStringify`), no MDX required | Confirmed via web search (see Web Citations): works in a plain `react-markdown` pipeline, same unified/rehype ecosystem already in use |
| `flexsearch` | Client-side search for `api-docs`, replacing Protocol's Algolia+FlexSearch combo | Apache-2.0 | Mature | Actively maintained | High — framework-agnostic, self-contained, light build is 4.5kb gzipped | Confirmed via its own README (see Web Citations): no Next.js/MDX dependency; supports build-time index serialization + client-side query, exactly the static-site shape needed |
| Algolia (`@algolia/autocomplete-core`) | Protocol's hosted search UI | Proprietary/commercial | N/A | N/A | Rejected | A paid, hosted SaaS search service — disproportionate for a single-owner site; FlexSearch alone (client-side, no hosted service) covers the same need at zero ongoing cost, consistent with Principle I |

**Selected:** React Router v7 resource routes + `feed` for `changelog`'s
RSS; `rehype-pretty-code`/Shiki for `api-docs` syntax highlighting;
`flexsearch` alone (no Algolia) for `api-docs` search.
**Rejected (with reason):** Algolia — disproportionate hosted-service cost
for a single-owner site; see table above. Next.js route handlers — wrong
framework target, already established in 001.
**Open questions:** see below.

## Architecture Patterns

### `broadcast` (Transmit-inspired)

- **Frontmatter:** `title`, `episode_number`, `publish_date`, `summary`,
  `audio_url` (or `embed_url` for an external host like Spotify/
  YouTube — see Open Questions), body Markdown for show notes.
- **Route structure:** mirrors Transmit's own shape — an index/list page
  plus a `[episode]`-style dynamic route, both prerenderable in React
  Router v7 (`prerender` can enumerate dynamic params from the content
  directory at build time, the same way biblio's existing types already
  generate per-entry static pages).
- **No RSS needed** for this type specifically (that's `changelog`'s
  signature feature) — `broadcast` is closer to a static catalog/archive.

### `changelog` (Commit-inspired)

- **Frontmatter:** `title`, `date`, `summary`, body Markdown for the
  entry. Directly comparable to biblio's existing `log` type's `entries/
  YYYY-MM-DD-slug.md` convention — the gap from `log` is purely the RSS
  feed and any Commit-style visual treatment, not the content model.
- **RSS feed**: a React Router v7 **resource route** (e.g.
  `routes/feed[.]xml.ts`) exporting only a `loader` that reads the
  content index (already generated at build time, per biblio's existing
  pipeline) and returns a `feed`-serialized `Response` with
  `Content-Type: application/rss+xml` — no Next.js route handler, no new
  build step beyond what biblio's content-index generation already does.

### `api-docs` (Protocol-inspired)

- **Frontmatter:** `title`, `order`, `section`, `summary` — **identical
  to biblio's existing `docs` type's confirmed schema**, reused as-is
  rather than inventing a new shape, since the underlying need (per-topic
  hierarchical nav) is the same.
- **Syntax highlighting:** add `rehype-pretty-code` (or `react-shiki`) to
  the existing `react-markdown`/`remark-gfm` pipeline as an additional
  rehype plugin — no MDX, no separate code-block component system.
- **Search:** build-time-generated FlexSearch index (page
  titles/headings/body text from the same content the prerender step
  already processes), shipped as a static JSON asset, queried entirely
  client-side — no search backend, no live calls, consistent with this
  category's backend-independent nature.

**Tradeoffs:** Shiki-based highlighting is more visually faithful (real
VS Code-grade themes) but has a larger bundle/build-time cost than a
lighter highlighter (e.g. Prism) — flagged in Open Questions, not decided
here. FlexSearch's index is generated at build time and shipped to the
client, so search quality is bounded by what fits reasonably in a static
JSON file — fine at personal-site scale, would need revisiting only if
content volume grew dramatically (an explicit, deferred escalation, same
posture as biblio's own search-escalation precedent).
**Integration points:** all three types still go through the same
content-index generation step biblio's existing types already use
(`content-index.json`) — the new types add fields to that pipeline, they
don't need a parallel one.
**Data model implications:** `api-docs` reuses `docs`'s existing schema
exactly — no new data model there. `broadcast` and `changelog` each add a
small, type-specific frontmatter shape, consistent with the per-type
fixed-schema convention biblio's own Decision Log already established
(no shared catch-all schema across types).

## Existing Codebase Evaluation

No `skipper` web-layer scaffolding code exists yet. biblio's existing
`docs` type frontmatter and content-index pipeline are the direct,
reusable precedent for `api-docs`; biblio's `log` type's
`entries/YYYY-MM-DD-slug.md` convention is the direct precedent for
`changelog`'s content layout.

## Security & Compliance

Unchanged from [001](001-web-layer-templates.md#security--compliance):
all three types are still original implementations of patterns observed
in Tailwind Plus-licensed examples, never copied source. No new secrets/
credentials surface — these remain backend-independent static sites.

## Performance, Reliability & Operations

- RSS and the search index are both generated **at build time**, not
  per-request — consistent with this category's fully static, prerendered
  nature. No live resource-route computation cost at request time beyond
  serving a pre-built response.
- Shiki's build-time cost (highlighting at build, not runtime, per
  `rehype-pretty-code`'s own design) keeps the `api-docs` type's runtime
  bundle lean regardless of which highlighter is chosen — the bundle-size
  tradeoff flagged above is about the *client-side theme/runtime* footprint,
  not build performance.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Building all three new types at once, before any real content exists for any of them, is speculative scope | Medium | Medium | Phase/prioritize via Open Questions below rather than committing to all three simultaneously | mona |
| FlexSearch's static, build-time index doesn't scale if `api-docs` content grows large | Low (personal-site scale) | Low–Medium | Explicitly deferred escalation — revisit only if real content volume demonstrates a gap, mirroring biblio's own search-escalation precedent | mona |
| Shiki's bundle/runtime footprint is heavier than necessary if visual fidelity isn't actually valued | Low–Medium | Low | Flagged in Open Questions — confirm Shiki vs. a lighter highlighter before implementation | mona |

## Open Questions

All three questions originally listed here are resolved — see the
Decision Log.

- [x] ~~Broadcast audio hosting?~~ — **Both supported**: frontmatter
  allows either `audio_url` (self-hosted, served from Cloudflare) or
  `embed_url` (external — Spotify/YouTube/etc.) per episode.
- [x] ~~Phasing?~~ — **Phased, not bundled**: `changelog` first (closest
  gap to the existing `log` type), then `api-docs`, then `broadcast`.
- [x] ~~Syntax highlighter?~~ — **Shiki**, via `rehype-pretty-code`.

## Decision Log

- **2026-06-24** — **`broadcast`'s frontmatter supports both
  `audio_url` (self-hosted) and `embed_url` (external embed) per
  episode, not just one.** — Rationale: keeps the type flexible across
  however a given episode's audio actually ends up hosted, without
  forcing a single distribution choice into the schema. — Decided by:
  mona.
- **2026-06-24** — **The three new types ship phased, not as one bundled
  feature: `changelog` first, then `api-docs`, then `broadcast`.** —
  Rationale: `changelog` has the smallest gap to close (closest to the
  existing `log` type's content model), so it's the lowest-risk first
  step; avoids committing design/implementation effort to all three
  before any real content exists for any of them (Principle I). —
  Decided by: mona.
- **2026-06-24** — **`api-docs` uses Shiki (via `rehype-pretty-code`) for
  syntax highlighting, not a lighter alternative.** — Rationale:
  highlighting happens at build time, not runtime, so Shiki's higher
  fidelity doesn't cost page-load performance — the bundle-size tradeoff
  noted in Risks is about the highlighter's own client-side footprint,
  which is an acceptable cost for the visual quality gained. — Decided
  by: mona.

## Internal References

- [001-web-layer-templates.md](001-web-layer-templates.md) — the policy
  decisions (new types, no-MDX, no-Catalyst, TypeScript-only) this doc
  implements concretely
- [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md) — the `skipper site new` scaffold command and Cloudflare deployment mechanism this doc's three types are generated and deployed through
- [roadmap.md §0](roadmap.md#0-outstanding-design-questions) — tracking
  location for this open item
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias), relevant to the phasing question

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| React Router — Resource Routes | https://reactrouter.com/how-to/resource-routes | 2026-06-24 | Confirms a route becomes a resource route (returns a raw `Response`, e.g. RSS/XML) by exporting a `loader` with no default component export — the direct replacement for Commit's Next.js `feed.xml/route.ts`. |
| `rehype-pretty-code` (Rehype Pretty) | https://rehype-pretty.pages.dev/ | 2026-06-24 | Confirms Shiki-powered syntax highlighting integrates as a rehype plugin in a plain `remark`/`rehype` pipeline — no MDX required — the mechanism `api-docs` reuses for code blocks. |
| `nextapps-de/flexsearch` README | https://github.com/nextapps-de/flexsearch | 2026-06-24 | Confirms FlexSearch is framework-agnostic and self-contained (no Next.js/MDX dependency), supports build-time index serialization + client-side query, and lists bundle sizes (4.5kb–16.3kb gzipped depending on build) — basis for dropping Algolia and using FlexSearch alone. |

## Appendix

None yet.
