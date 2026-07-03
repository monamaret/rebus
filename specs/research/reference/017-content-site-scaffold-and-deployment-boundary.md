# Research: Standalone content-site scaffolding and the deployment boundary

**Status:** Complete — scaffold/lifecycle mechanism resolved by wrapping biblio-cli directly, corrected from an earlier Cloudflare-native-integration framing; ready to update specs/research/001-web-layer-templates.md and specs/research/008-new-content-site-types.md
**Date:** 2026-06-24
**Owner:** mona

**Correction note (2026-06-24):** This doc originally selected
Cloudflare Workers Builds' native Git integration as the deployment
mechanism, reasoning that `skipper`'s involvement ends at scaffolding
with nothing further for any tool to manage. The owner corrected this:
sites scaffolded by `skipper` must be **maintained, deployed, updated,
and managed by `biblio-cli` itself** — not left to a bare Cloudflare
auto-deploy with no lifecycle tooling. Verifying `biblio-cli`'s actual
documented behavior (see Web Citations) showed it already owns this
exact lifecycle end to end (`create → configure → update → build →
deploy`, including its own generated CI workflow that runs `biblio-cli
build && biblio-cli deploy` on push) — so the correct architecture is
`skipper` **wrapping `biblio-cli`** for scaffolding (Principle II:
orchestrate, don't reimplement), with `biblio-cli` itself — not
`skipper`, and not bare Cloudflare tooling — owning everything
afterward. The sections below are rewritten accordingly; superseded
reasoning is removed rather than kept side-by-side, per this skill's own
convention of recording confirmed decisions, not a history of wrong
turns.

## Problem Statement

[010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md)
and [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md)
each resolved the concrete deployment mechanics for one of `skipper`'s
two GCP-targeting deployment categories (GKE backends; Firebase Hosting/
Cloud Run for the generic CLI+web scaffold) — but the platform's *third*
deployment target, **Cloudflare Workers for standalone content sites**
([001-web-layer-templates.md](001-web-layer-templates.md)/
[008-new-content-site-types.md](008-new-content-site-types.md)'s
category), never had the same treatment. Unlike the other two
categories, this one already has its own dedicated lifecycle tool —
`biblio-cli` — so the real question isn't "what does `skipper` build to
manage this," it's "how does `skipper` hand off to the tool that already
manages this correctly."

- **Goal:** Confirm how `skipper`'s scaffold command relates to
  `biblio-cli`, and confirm that ongoing maintenance/deployment/updates
  for a scaffolded site are `biblio-cli`'s job, not `skipper`'s or a
  bespoke Cloudflare-only mechanism — narrow enough to name a concrete
  `skipper` command and update `001`/`008`.
- **In scope:** The scaffold command's shape and its relationship to
  `biblio-cli new`; confirming `biblio-cli`'s own lifecycle commands
  (`configure`/`update`/`build`/`deploy`) are what manages a scaffolded
  site afterward; the resulting CI/credential shape.
- **Out of scope:** Re-opening `001`/`008`'s settled content/policy
  decisions (site types, frontmatter schemas, no-MDX, TypeScript-only) —
  not reconsidered here. Redesigning `biblio-cli` itself — it's an
  external reference tool `skipper` orchestrates, not code this repo
  owns or modifies.
- **Success criteria:** A confirmed scaffold-command shape and lifecycle
  ownership, recorded in this doc's Decision Log.
- **Non-goals:** Designing the actual scaffold template's file contents
  in further detail than `008` already did (frontmatter, routes,
  content-index pipeline) — this doc is about the command/lifecycle
  *mechanism* around that already-designed content, not the content
  itself.

## Background & Context

- **Relevant prior work:**
  - [001 → Decision Log](001-web-layer-templates.md#decision-log) — confirms Cloudflare Workers as this category's deployment target, biblio's stack (React Router v7 native `prerender`) as the reference.
  - [010 → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log) and [012 → Decision Log](012-firebase-hosting-cloud-run-deployment.md#decision-log) — the analogous mechanics docs this one mirrors in structure for the third deployment target.
  - [.specify/memory/constitution.md Principle VIII](../../.specify/memory/constitution.md) — standalone content sites are explicitly the *backend-independent* category, distinct from GCP-native backend-application UIs.
  - [.specify/memory/constitution.md Principle II](../../.specify/memory/constitution.md) — "`skipper` orchestrates and wraps underlying tools... it MUST NOT reimplement their functionality" — directly resolves this doc's core question: `biblio-cli` is exactly such an underlying tool for this category, the same role `kubectl`/`gcloud`/`firebase`/`wrangler` play for the other two.
- **Domain assumption:** `biblio-cli` (`github.com/monamaret/biblio`) is
  not just a reference pattern to draw inspiration from — it is a real,
  installable Go CLI that already owns the full lifecycle for exactly
  this category of site (per its own documentation, see Web Citations).
  `skipper` should use it directly, the same way it uses `kubectl`/
  `gcloud`/`firebase` directly elsewhere, rather than re-deriving
  equivalent scaffold/lifecycle logic of its own.
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **`biblio-cli`** (`github.com/monamaret/biblio`) — `new`/`index`/`dev`/`build`/`deploy`/`update` | Full content-site lifecycle: scaffold, author, build, deploy, and keep scaffold-tracked parts current | Owner's own tool (not third-party) | Documented, working CLI with a recorded release lifecycle (v0.06 docs) | Maintained by mona | High | Confirmed via `biblio-cli`'s own README and v0.06 lifecycle docs (see Web Citations): "the whole lifecycle — create, configure, preview, build, deploy, update — is meant to be driven through `biblio-cli`." `skipper` wrapping this directly is the correct application of Principle II — it's exactly the kind of underlying tool `skipper` is meant to orchestrate, not reimplement. |
| `skipper` reimplementing scaffold/lifecycle logic of its own for this category | An in-house alternative to `biblio-cli` | N/A (would be in-house) | N/A | N/A | Rejected | Would duplicate a tool that already exists, is documented, and already correctly handles this exact lifecycle — a direct Principle II violation, not just a missed opportunity for reuse. |
| Cloudflare Workers Builds' native Git integration (considered in an earlier draft of this doc) | Bare auto-deploy-on-push, bypassing `biblio-cli` entirely | Cloudflare's own managed CI | Mature | Cloudflare-maintained | Rejected (superseded) | Would deploy a scaffolded site *around* `biblio-cli` instead of through it, losing `biblio-cli update`'s scaffold-tracked-file reconciliation (`.biblio/scaffold-manifest.json` diffing) and the single consistent lifecycle surface the owner explicitly wants. Not used. |

**Selected:** `skipper site new --type=<type> <repo-name>` directly wraps
`biblio-cli new` to scaffold a site; every subsequent lifecycle action
(authoring, `index`, `update`, `build`, `deploy`) runs through
`biblio-cli` itself, exactly as `biblio-cli`'s own documentation
describes, whether invoked by the owner by hand or by `biblio-cli`'s own
generated CI workflow.
**Rejected (with reason):** an in-house `skipper` scaffold/lifecycle
implementation (duplicates an existing, working tool); bare Cloudflare
native-integration deploy bypassing `biblio-cli` (loses `biblio-cli
update`'s reconciliation and the single-lifecycle-surface goal). See
table.
**Open questions:** none for this section — resolved directly from
`biblio-cli`'s own documented lifecycle and Principle II, not an owner
preference call.

## Architecture Patterns

- **`skipper site new --type=<type> <repo-name>` is a thin wrapper around
  `biblio-cli new`, nothing more.** It shells out to (or directly
  invokes, if linked as a library — an implementation detail, not a
  research blocker) `biblio-cli new <repo-name> --type=<type>
  --domain=<domain> --theme=<theme>`, generating a new, separate
  repository per constitution Principle VIII (never inside `skipper`'s
  own repo). `--type` covers biblio's existing types (`profile`/`log`/
  `docs`) and the three new ones `008` confirmed (`broadcast`/
  `changelog`/`api-docs`), assuming `biblio-cli` itself is extended with
  those types (a `biblio-cli`-side concern, not `skipper`'s — see Out of
  scope). `skipper` adds no scaffold logic of its own beyond constructing
  this one invocation.
- **`biblio-cli` owns the entire lifecycle after scaffolding — `skipper`
  has no further role.** Per `biblio-cli`'s own documented lifecycle:
  `create` (the `new` command above) produces a `.biblio/scaffold-
  manifest.json` baseline; the owner authors content and runs `biblio-cli
  index` to regenerate the content index; `biblio-cli update` brings
  scaffold-tracked parts (`workers/`, `app/`, build config, **the CI
  workflow itself**) up to date against that baseline, halting on
  conflicts rather than silently overwriting customizations; `biblio-cli
  build` runs the `content-index → React Router prerender → wrangler
  bundle` pipeline; `biblio-cli deploy` runs `wrangler deploy` against the
  route pre-filled from `--domain` in the generated `wrangler.jsonc`. The
  exact same `build`/`deploy` pair runs whether invoked by hand or by the
  CI workflow `biblio-cli new` itself scaffolds into the repo — "the same
  two commands on push," per `biblio-cli`'s own documentation — which is
  precisely the single, consistent lifecycle surface the owner wants
  preserved.
- **No `skipper site update`/`upgrade`/`deploy` command is built.**
  `biblio-cli update`/`build`/`deploy` already exist and already do this
  correctly — adding a `skipper`-level equivalent would be redundant
  indirection over a tool that's already a clean, direct CLI, not a
  missing capability. This is a narrower, corrected version of an earlier
  draft's "`skipper`'s role ends at scaffolding, nothing manages it
  afterward" — the role does end at scaffolding, but something *does*
  manage it afterward: `biblio-cli`, by design.

**Tradeoffs:** None significant — wrapping an existing, already-correct
tool rather than reimplementing equivalent logic is close to a pure win
under Principle II; the only real cost is `skipper site new` depending on
`biblio-cli` being installed/available, an accepted, already-implicit
dependency given `biblio-cli` is this category's named reference tool
throughout this platform's research.
**Integration points:** None beyond the one scaffold invocation — no
shared deployment mechanism with `010`/`012`'s categories, since
`biblio-cli`, not `skipper`, owns this category's lifecycle end to end.
**Data model implications:** None on `skipper`'s side — `.biblio/
scaffold-manifest.json` is `biblio-cli`'s own state, living in each
scaffolded site's own repo, never in `skipper`'s.

## Existing Codebase Evaluation

No `skipper` scaffold-command code exists yet. `biblio-cli` is not just a
loose stylistic precedent here — it is the literal tool `skipper site
new` wraps, and the literal tool that manages everything afterward. No
separate `skipper`-side lifecycle implementation is warranted.

## Security & Compliance

- `biblio-cli deploy` runs `wrangler deploy` directly (confirmed via its
  own v0.06 lifecycle docs, see Web Citations) — this requires Wrangler's
  own Cloudflare authentication (an API token), stored as a CI secret in
  each scaffolded site's own repo for its generated CI workflow to use.
  Cloudflare has no OIDC/WIF-equivalent keyless alternative for Wrangler
  deploys as of this research (confirmed via Cloudflare's own community
  discussion, see Web Citations) — this is an accepted, currently
  unavoidable static credential, scoped to each individual site's own
  repo (not `skipper`'s, and not shared across sites), consistent with
  how `biblio-cli` already operates everywhere else it's used.
- No new credential type is introduced *by `skipper`* — the token lives
  in `biblio-cli`'s own generated CI workflow, in the scaffolded site's
  own repo, exactly as it would whether or not `skipper` was involved in
  scaffolding at all.

## Performance, Reliability & Operations

- Build/deploy reliability for a scaffolded site is `biblio-cli`'s own
  responsibility (its `build`/`deploy` commands and their CI invocation)
  — no `skipper`-side failure mode to design around, since `skipper`
  isn't in the lifecycle path at all after the initial `new` invocation.
- `biblio-cli update`'s conflict-halting behavior (stopping rather than
  silently overwriting when scaffold-tracked files have been customized)
  is `biblio-cli`'s own existing safety mechanism — `skipper` doesn't
  need to add anything on top of it.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| `skipper site new` is invoked on a machine without `biblio-cli` installed | Low–Medium | Low | `skipper` checks for `biblio-cli` on `PATH` before invoking it and fails with a clear, actionable error (install instructions) rather than a confusing exec failure | mona |
| `biblio-cli` doesn't yet support the three new types (`broadcast`/`changelog`/`api-docs`) confirmed in `008`, since those were designed in `skipper`'s research, not yet built into `biblio-cli` itself | Medium | Medium | Out of scope for `skipper` to fix — `biblio-cli` needs its own update to add these `--type` values before `skipper site new --type=changelog` can work; tracked as a dependency, not designed here (see Out of scope) | mona |
| Cloudflare later ships OIDC/"trusted publishing" support for Wrangler (already requested, per Web Citations) | Low–Medium | Low | Would be `biblio-cli`'s own change to adopt, not `skipper`'s — revisit only if `biblio-cli` itself is updated to use it | mona |

## Open Questions

None for `skipper`'s own scope — the wrapping relationship and lifecycle
ownership are resolved directly above. One real dependency is flagged in
Risks, not an open research question: `biblio-cli` itself needs to
support the three new `008` site types before `skipper site new` can
scaffold them — that's `biblio-cli`'s own backlog item, tracked here as a
dependency rather than designed in this `skipper`-side doc.

## Decision Log

- **2026-06-24** — **`skipper site new --type=<type> <repo-name>` wraps
  `biblio-cli new` directly — `skipper` implements no scaffold logic of
  its own for this category.** — Rationale: `biblio-cli` already exists,
  is documented, and correctly does this; reimplementing it would
  directly violate Principle II. — Decided by: mona.
- **2026-06-24** — **Every lifecycle action after scaffolding
  (authoring, `index`, `update`, `build`, `deploy`) is `biblio-cli`'s job,
  not `skipper`'s — no `skipper site update`/`upgrade`/`deploy` command is
  built.** — Rationale: `biblio-cli` already owns this exact lifecycle,
  including its own CI workflow running the same `build`/`deploy` pair on
  push; a `skipper`-level equivalent would be redundant indirection over
  an already-clean tool. — Decided by: mona.
- **2026-06-24** — **Deployment goes through `biblio-cli deploy`
  (`wrangler deploy`), accepting the Cloudflare API token it requires as
  a CI secret in each scaffolded site's own repo** — not Cloudflare's
  native Git integration, which would bypass `biblio-cli` and lose its
  `update` reconciliation. — Rationale: consistency with `biblio-cli`'s
  single, already-correct lifecycle surface outweighs avoiding a static
  credential Cloudflare doesn't yet offer a keyless alternative to
  anyway. — Decided by: mona.

## Internal References

- [001-web-layer-templates.md](001-web-layer-templates.md) — the Cloudflare Workers deployment-target decision this doc fills in the mechanism for
- [008-new-content-site-types.md](008-new-content-site-types.md) — the frontmatter/content-pipeline design this doc's scaffold command generates, pending `biblio-cli`'s own support for the new types
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) and [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md) — the analogous prior docs this one mirrors in structure
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle II (orchestrate, don't reimplement — directly decisive here), Principle VI (secure-by-default), and Principle VIII (repo separation, backend-independent content category)

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `github.com/monamaret/biblio` — README | https://github.com/monamaret/biblio | 2026-06-24 | Confirms `biblio-cli`'s command surface (`new`, `index`, `dev`, `build`, `deploy`) and that "the whole lifecycle — create, configure, preview, build, deploy, update — is meant to be driven through `biblio-cli`" — the basis for wrapping it directly rather than reimplementing equivalent logic. |
| `github.com/monamaret/biblio` — v0.06 lifecycle docs (`docs/releases/v0.06/README.md`) | https://github.com/monamaret/biblio/blob/main/docs/releases/v0.06/README.md | 2026-06-24 | Confirms the concrete `create → configure → update → build → deploy` lifecycle: `new` produces a `.biblio/scaffold-manifest.json` baseline; `update` reconciles scaffold-tracked files against it, halting on conflict; `build` runs `content-index → react-router prerender → wrangler bundle`; `deploy` runs `wrangler deploy` to the route in `wrangler.jsonc`; the same `build`/`deploy` pair runs on push via `biblio-cli`'s own generated CI workflow. This is the basis for this doc's corrected Decision Log. |
| `cloudflare/workers-sdk` — Discussion #11434, "OIDC / 'trusted publishing' support for wrangler deploys" | https://github.com/cloudflare/workers-sdk/discussions/11434 | 2026-06-24 | Confirms OIDC/keyless deploys for Wrangler are requested but not yet available as of this research — basis for accepting the static Cloudflare API token `biblio-cli deploy` requires. |

## Appendix

None yet.
