# Research: Web application layer — new scaffold templates from Catalyst + Tailwind Plus examples

**Status:** Complete — all open questions resolved; ready for feature-item creation
**Date:** 2026-06-24
**Owner:** mona

**Scope note (added 2026-06-24):** The web application layer now splits
into two categories (constitution Principle VIII):
**standalone, backend-independent content sites** (this doc's scope —
biblio's Cloudflare Workers pattern, profile/log/docs and the new
templates explored below) and **backend-application UIs** (a thin client
wired to the GCP/GKE backend, hosted GCP-natively on Firebase Hosting +
Cloud Run — see [005-generic-app-scaffold.md](005-generic-app-scaffold.md)).
Cloudflare Workers remains the right target for *this* doc's category;
the GCP-native decision in 005 does not reverse anything here.

**Scope note (added 2026-06-24):** [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md)
resolved how sites in this category are actually scaffolded and managed:
`skipper site new --type=<type> <repo-name>` (generating a new, separate
repo per constitution Principle VIII) directly wraps `biblio-cli new` —
`skipper` implements no scaffold logic of its own. Every lifecycle action
afterward (authoring, `index`, `update`, `build`, `deploy`) is
**`biblio-cli`'s job**, including `biblio-cli deploy` running `wrangler
deploy` via `biblio-cli`'s own generated CI workflow — `skipper` has no
ongoing role and builds no `site update`/`deploy` command, since
`biblio-cli` already provides one.

## Problem Statement

`skipper`'s web application layer builds on `github.com/monamaret/biblio`'s
scaffolding pattern (Go CLI generating a React + Vite + React Router v7 +
Cloudflare Worker project per site, currently three site types: `profile`,
`log`, `docs`). New UI assets have been dropped into `specs/assets/` to use
as inspiration for additional templates and for upgrading the existing
ones: the Catalyst UI Kit (an application-UI component library) and three
Tailwind Plus example sites (Transmit, Commit, Protocol). None of these are
to be copied directly — they are inspiration for `skipper`'s own scaffold
templates, adapted to biblio's actual stack (Vite/React Router v7/Cloudflare
Workers, not Next.js/Vercel).

- **Goal:** Determine which new web-layer template(s) and/or component
  upgrades these assets justify, and how each maps onto (or diverges from)
  biblio's existing site-type pattern and stack.
- **In scope:** Cataloging what each asset actually is, how its stack
  compares to biblio's, and which parts are genuinely reusable as scaffold
  inspiration vs. Next.js-specific dead ends.
- **Out of scope:** Vendoring or directly copying any example site's code;
  generating dummy/placeholder assets for every example up front.
- **Success criteria:** A clear, owner-reviewed mapping from
  asset → candidate template type → stack adaptation needed, recorded in
  this doc's Decision Log, feeding feature items and a roadmap entry.
- **Non-goals:** Deciding final visual design or full component API surface
  in this document — that's implementation detail once a feature item
  exists.

## Background & Context

- **Relevant prior work:** `specs/research/architecture.md` already
  confirms the web layer's deployment target is Cloudflare Workers via a
  React framework scaffold, with `biblio` as the named reference
  implementation. `specs/research/roadmap.md` §0 already flags "web
  application layer stack specifics" as an open question this research
  doc resolves.
- **Domain assumptions:** biblio's existing three site types (`profile`,
  `log`, `docs`) are themselves built from Tailwind Plus-style design
  sensibilities but hand-rolled, not built from these specific kits.
  biblio deliberately avoids MDX (plain Markdown + YAML frontmatter via
  `gray-matter`, per its own Decision Log) — **resolved (see Decision
  Log): that constraint carries over unchanged, plain Markdown, no MDX**,
  even though two of the three new example assets are MDX-based.
- **Stakeholders:** mona (sole owner/operator of the resulting platform).

## Libraries & Stack

Contents of `specs/assets/` (already unzipped) and what each one is:

| Asset | What it is | Stack | License surface |
| --- | --- | --- | --- |
| `catalyst-ui-kit/` | Application UI component library (not a full site) — buttons, dialogs, forms, tables, sidebar/stacked layouts, navbar, pagination, etc., in both TS and JS variants, plus a demo "events/teams/users" admin dashboard app | React 19, Tailwind CSS v4, Headless UI, Motion (Framer Motion), `clsx`; demo app is Next.js | Tailwind Plus commercial license — components are meant to be copied into a project, not installed as a package |
| `transmit/` | Tailwind Plus example site — single-page + `[episode]` dynamic route; podcast/episode-style marketing site | Next.js 16, React 19, React Aria/Stately, `valibot`, no MDX | Tailwind Plus commercial license |
| `commit/` | Tailwind Plus example site — blog/changelog with MDX content pages, RSS feed route (`feed.xml/route.ts`), Motion-based transitions | Next.js 16, MDX (`@next/mdx`), `feed`, `next-themes`, Motion | Tailwind Plus commercial license |
| `protocol/` | Tailwind Plus example site — API reference docs with per-topic MDX pages (quickstart, authentication, pagination, webhooks, SDKs, etc.), Algolia/FlexSearch-based search, syntax highlighting | Next.js 16, MDX, `@headlessui/react`, `flexsearch`, `@algolia/autocomplete-core`, Framer Motion | Tailwind Plus commercial license |

Comparison against biblio's confirmed stack (React + Vite + React Router v7
native `prerender`, Tailwind CSS, shadcn/ui on Radix, plain Markdown +
`gray-matter` + `react-markdown`/`remark-gfm`, deployed as a Cloudflare
Worker):

- **Framework mismatch:** all three example sites and the Catalyst demo are
  Next.js (App Router, route handlers, `next/mdx`), not Vite/React Router
  v7. None of the routing, MDX pipeline, or RSS-route code transfers
  directly — every example's *page structure and visual/component design*
  is the reusable part, not its Next.js plumbing.
- **Component layer compatibility:** Catalyst's components are plain React +
  Tailwind v4 + Headless UI + Motion + `clsx` — no Next.js-specific APIs in
  the component files themselves (only the demo app wrapping them is
  Next.js). These are framework-agnostic enough to drop into a Vite/React
  Router v7 project largely as-is, modulo `shadcn/ui`/Radix vs.
  Headless UI/Motion overlap that needs reconciling (biblio currently uses
  shadcn/ui on Radix, not Headless UI — using both in one project is
  redundant).
- **Content pipeline mismatch:** Commit and Protocol both rely on MDX
  (`@mdx-js/*`, `@next/mdx`) for content authoring. biblio's confirmed
  Decision Log explicitly rejected MDX in favor of plain Markdown +
  frontmatter. Adopting Commit/Protocol's patterns wholesale would require
  either reversing that decision for `skipper`'s templates or re-deriving
  their UI (search, syntax highlighting, per-topic nav) on top of plain
  Markdown instead.
- **Selected:** *(pending Decision Log entries below)*
- **Rejected (with reason):** Direct adoption of any example site's Next.js
  routing/build layer — wrong framework target for Cloudflare Workers
  deployment via biblio's pattern.
- **Open questions:** see below.

## Architecture Patterns

Candidate template/feature mappings, by asset:

- **Catalyst UI Kit → component upgrade across all site types.** Its
  layouts (`sidebar-layout`, `stacked-layout`, `navbar`, `auth-layout`) and
  primitives (forms, tables, dialogs, dropdowns) are aimed at *application*
  UIs (the demo is an admin dashboard), not content/marketing sites — this
  is a stronger fit for any future "app" template (a real interactive web
  application, as opposed to biblio's current static/content site types)
  than for `profile`/`log`/`docs` as they exist today.
- **Transmit → candidate new template** for an episodic/broadcast-style
  site (podcast, newsletter archive). Closest existing biblio type: none —
  this would be a genuinely new site type.
- **Commit → candidate upgrade or new template** for the `log` site type —
  Commit's RSS feed route and changelog-entry layout overlap heavily with
  what `log` already is conceptually; the gap is MDX vs. plain Markdown and
  Next.js vs. Vite/RRv7.
- **Protocol → candidate upgrade or new template** for the `docs` site
  type — Protocol's per-topic nav, search, and syntax highlighting are a
  meaningfully richer docs experience than biblio's current `docs` type;
  same MDX/Next.js gap applies.
- **Tradeoffs:** upgrading `log`/`docs` in place vs. adding new sibling
  template types is itself an open question — see below.
- **Integration points:** any Catalyst component adoption needs a decision
  on Headless UI + Motion vs. biblio's existing shadcn/ui + Radix, not both.
- **Data model implications:** adopting MDX for `log`/`docs` (if decided)
  would change biblio's confirmed content-authoring model; needs its own
  Decision Log entry here rather than being assumed.

## Existing Codebase Evaluation

`skipper` itself has no web-layer scaffolding code yet — this is greenfield
within `skipper`, building on the *pattern* established in `biblio`
(external reference repo, not vendored into `skipper`). No refactor/reuse
assessment applies yet beyond what's captured above.

## Security & Compliance

- All four assets are Tailwind Plus **commercial-licensed** products
  (`tailwindcss.com/plus`). Per the user's instruction, none of their code
  is to be copied directly into generated scaffolds — they are inspiration
  only (visual/structural patterns), not source to redistribute. Generated
  scaffold templates must be original implementations of the *patterns*
  observed, not derivative copies of licensed source files.
- No secrets, API keys, or credentials present in the asset directories
  themselves (not checked in detail per-file, but none of these are
  expected to need backend credentials at the template stage).

## Performance, Reliability & Operations

Not yet applicable — no implementation exists. Revisit once a template's
build output is measured against biblio's existing Cloudflare Workers
deploy size/performance baseline (if one exists).

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Accidentally porting Tailwind Plus-licensed source verbatim instead of re-implementing the pattern | Medium | High (licensing) | Treat `specs/assets/*` strictly as read-only reference; new template code is written fresh against biblio's stack, never copy-pasted | mona |
| Adopting MDX for `log`/`docs` silently reverses biblio's no-MDX decision without a recorded rationale | Medium | Medium | Require an explicit Decision Log entry here (or a superseding one in biblio if that repo is also touched) before any MDX dependency is added | mona |
| Running both shadcn/ui+Radix and Headless UI+Motion in one generated project | Medium | Low–Medium (bundle size, inconsistent UI primitives) | Decide one component-primitive stack per template before implementation | mona |

## Open Questions

All questions originally listed here are resolved — see the Decision Log.

## Decision Log

- **2026-06-24** — `skipper`'s new/upgraded web-layer templates stay on
  plain Markdown + frontmatter, matching biblio's existing content model;
  Commit's and Protocol's MDX usage is *not* adopted. Any search,
  syntax-highlighting, or per-topic nav UX inspired by Commit/Protocol is
  re-derived on top of plain Markdown rather than pulling in `@mdx-js/*`/
  `@next/mdx`. — Rationale: keeps one consistent content-authoring model
  across all site types rather than splitting plain-Markdown types from
  MDX types; avoids reversing biblio's own prior decision without a
  stronger justification than "the example happened to use MDX." — Decided
  by: mona.
- **2026-06-24** — **Transmit, Commit, and Protocol all become new sibling
  site types — not upgrades to the existing `log`/`docs` types.** —
  Rationale: keeps the existing `profile`/`log`/`docs` types untouched and
  stable while adding the new patterns as their own types, avoiding a
  forced in-place migration of existing sites. — Decided by: mona.
- **2026-06-24** — **Catalyst UI Kit stays scoped to the generic app
  scaffold ([R-005](005-generic-app-scaffold.md)); the standalone-
  content-site category keeps shadcn/ui-on-Radix.** — Rationale: Catalyst
  is application-shaped (dashboards, forms, tables); biblio's content
  sites are read-oriented — a poor fit. Avoids running two component
  primitive stacks (shadcn/ui+Radix and Headless UI+Motion) in one
  category. — Decided by: mona.
- **2026-06-24** — **Content-site templates are TypeScript-only; no
  JavaScript variant is generated.** — Rationale: matches biblio's existing
  stack and constitution Principle III's idiomatic-TypeScript mandate; no
  second variant to maintain for a single-developer platform. — Decided
  by: mona.

> **Format note (2026-07-03):** this doc predated the current
> `research-template.md`; its single `## References` list has been split into
> `## Internal References` + `## Web Citations` to match the template.

## Internal References

- [architecture.md](architecture.md) — confirms web layer = Cloudflare
  Workers via React framework scaffold, biblio as reference
- [roadmap.md §0](roadmap.md#0-outstanding-design-questions) — "web
  application layer stack specifics" open item this doc addresses
- [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md) — the scaffold command and deployment mechanism for this category
- `specs/assets/catalyst-ui-kit/` — local copy of the Catalyst UI Kit (component library)
- `specs/assets/transmit/`, `specs/assets/commit/`, `specs/assets/protocol/` — local copies of the Transmit/Commit/Protocol Tailwind Plus example sites

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `monamaret/biblio` | https://github.com/monamaret/biblio | 2026-06-24 | Reference implementation for the content-site scaffolding pattern and the current site types this doc extends. |
| Tailwind Plus (Catalyst UI Kit; Transmit/Commit/Protocol templates) | https://tailwindcss.com/plus | 2026-06-24 | Upstream source of the Catalyst UI Kit and the Transmit/Commit/Protocol example sites vendored under `specs/assets/` and evaluated here as content-site-type precedents. |

## Appendix

None yet.
