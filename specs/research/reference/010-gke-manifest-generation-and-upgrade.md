# Research: GKE manifest generation, persistence, and upgrade mechanics

**Status:** Complete — all questions resolved directly from existing constitutional requirements and the bbb-le precedent; ready to update specs/research/003-backend-application-deployment.md and dependent feature items
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[003-backend-application-deployment.md](003-backend-application-deployment.md)
and every feature item that deploys to GKE (off-the-shelf images, stash,
chat, the dashboard) all reference "a declarative GKE manifest
(Deployment, Service, etc.) per Principle IV defaults" as the deploy
target — but none of them define what that manifest-generation mechanism
actually *is*. `003`'s own Existing Codebase Evaluation already named
`bbb-le`'s Kustomize base + overlays (`deploy/k8s/`) as "the closest
existing reference for the declarative-manifest shape this pipeline
targets," but never confirmed Kustomize as the chosen mechanism, nor
addressed two load-bearing follow-on questions: where do the *generated*
manifests live (persisted and version-controlled, or computed and applied
on the fly with nothing left behind), and what do `skipper app
update`/`upgrade` (named in constitution Principle II's lifecycle —
scaffold → deploy → **upgrade** → maintain) actually do mechanically.

- **Goal:** Confirm the manifest-generation/templating mechanism, where
  generated manifests are persisted, and what `app update`/`app upgrade`
  concretely do — narrow enough to update `003-backend-application-deployment.md`
  and every dependent feature item with a real mechanism instead of a
  black box.
- **In scope:** Templating mechanism (Kustomize vs. Helm vs. raw
  Go-templated YAML vs. a full IaC tool); manifest persistence model
  (GitOps-style, committed in this repo, vs. ephemeral generate-and-apply);
  `update` vs. `upgrade` semantics.
- **Out of scope:** Re-opening `003`'s already-settled image-source
  decisions (Docker Hub direct, GitHub Actions + WIF, Cloud Build for
  local-dir) — this doc is about what happens *after* an image is ready,
  not how it gets built.
- **Success criteria:** A confirmed manifest-generation mechanism,
  persistence model, and update/upgrade semantics, recorded in this doc's
  Decision Log.
- **Non-goals:** Designing the exact per-app manifest field values
  (resource limits, probe thresholds) — Principle IV already establishes
  the production-sensible defaults; this doc is about the generation
  *mechanism*, not re-litigating those defaults.

## Background & Context

- **Relevant prior work:**
  - [003 → Existing Codebase Evaluation](003-backend-application-deployment.md) — names `bbb-le`'s `deploy/k8s/` (Kustomize base + `local`/`gke`/`production` overlays) as the closest existing reference, never confirmed as the actual choice.
  - [.specify/memory/constitution.md Principle II](../../.specify/memory/constitution.md) — names the full lifecycle `skipper` must support as a CLI surface: scaffold → deploy → **upgrade** → maintain.
  - [.specify/memory/constitution.md Principle IV](../../.specify/memory/constitution.md) — requires generated Kubernetes configuration be declarative and that `skipper`'s deployment/upgrade/maintenance commands "operate against this declarative source of truth (**GitOps-friendly**)" — directly bears on the persistence question below.
  - [.specify/memory/constitution.md Principle VIII](../../.specify/memory/constitution.md) — this repo (`skipper`) "holds... the backend infrastructure/deployment configuration it manages directly," which is exactly where persisted manifests would live if that's the chosen model.
  - `003`'s own Performance section already flagged `app update --image <new-ref>` as "likely... but not scoped here" — this doc is that scoping pass.
- **Domain assumption:** Unlike `bbb-le` (which supports three genuinely
  different deployment targets — local dev, GKE, production — hence three
  overlays), `skipper` targets a single GKE cluster per platform instance.
  A multi-overlay structure built for multi-target flexibility `skipper`
  doesn't need would be carrying over design weight from a reference repo
  built for a different scale (the same caution already applied when
  rejecting rook's `space-id` concept in `009`).
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **Kustomize** (via `kubectl apply -k`, no separate binary) | Manifest templating/overlay mechanism | Apache-2.0 | Mature — built into `kubectl` natively since v1.14 | Maintained as part of `kubectl`/Kubernetes itself | High | Confirmed via Kubernetes' own docs (see Web Citations): no extra tool to install or version-pin beyond `kubectl`, which `skipper` already orchestrates per Principle II. Directly matches the already-named `bbb-le` precedent. |
| Helm | Manifest templating via a packaging/release system | Apache-2.0 | Mature, widely adopted | Actively maintained (CNCF) | Rejected | Helm's release/values-file/chart-versioning model is real machinery for managing *many* parameterized installs of the *same* chart across many users/clusters — `skipper` generates one manifest set per app, for one owner, on one cluster; that machinery has no audience here (Principle I). |
| Raw Go-templated YAML (no Kustomize/Helm at all) | `skipper` renders plain YAML strings directly | N/A (in-house) | N/A | N/A | Rejected | Loses Kustomize's structured base+patch reuse for free (e.g. consistently applying Principle IV defaults across every app) in exchange for no real benefit — `skipper` would have to hand-roll the same patch/merge logic Kustomize already provides natively in `kubectl`. |
| cdk8s / Pulumi (full IaC programming model) | Programmatic manifest generation | Apache-2.0 / various | Mature | Actively maintained | Rejected | A full IaC language/toolchain is disproportionate to generating Deployment/Service manifests for a handful of apps on one cluster — adds a dependency and learning-surface Principle I doesn't justify here. |

**Selected:** Kustomize, via `kubectl apply -k` — no new dependency, no
new binary to install or version, directly reuses the already-named
`bbb-le` precedent.
**Rejected (with reason):** Helm, raw YAML, cdk8s/Pulumi — see table.
**Open questions:** none for this section — resolved directly from the
"built into `kubectl`, matches the existing precedent" reasoning, not an
owner preference call.

## Architecture Patterns

- **One Kustomize base per app, no environment overlays for v1.** Unlike
  `bbb-le`'s three overlays (local/gke/production — built for a tool
  supporting multiple real deployment targets), `skipper` targets exactly
  one GKE cluster per platform instance. A single base directory per app
  (`Namespace`, `Deployment`, `Service`, any `ConfigMap`/`Secret`
  references, and the Workload Identity binding resources —
  `ServiceAccount`, Config Connector's `IAMServiceAccount`/
  `IAMPolicyMember` — pre-filled with Principle IV/V's defaults; the
  `Namespace` and IAM resources' shared per-app naming convention
  `skipper-<app-name>` resolved in
  [019](019-gke-namespace-strategy.md#decision-log) and
  [021](021-iam-service-account-scoping-strategy.md#decision-log)
  respectively) is sufficient — no overlay layer is needed until a second
  real
  environment (e.g. a staging cluster) actually exists to overlay
  against.
- **Generated manifests are persisted and committed in this repo
  (GitOps-style), not computed-and-applied ephemerally.** Each app gets a
  directory (e.g. `deployments/<app-name>/`) holding its Kustomize base,
  generated by `skipper app add` and modified in place by `skipper app
  update`/`upgrade`. `skipper` then runs `kubectl apply -k
  deployments/<app-name>/` against that persisted, version-controlled
  source. This directly satisfies Principle IV's explicit "GitOps-
  friendly" requirement and Principle VIII's framing of this repo as the
  place backend deployment configuration lives — an ephemeral,
  never-persisted generate-and-apply model would satisfy neither.
- **`skipper app update --image <new-ref>`**: patches the `image:`
  field in the app's persisted Kustomize base (via a Kustomize image
  transformer, or a direct file edit — implementation detail) and
  re-applies. This is a config change to an already-deployed app.
- **`skipper app upgrade`**: distinct from `update` — covers changes to
  `skipper`'s own *generated manifest shape* itself (e.g. if a future
  `skipper` release changes what Principle IV's "production-sensible
  defaults" means, `upgrade` re-renders an existing app's base against
  the current template and re-applies, similar in spirit to how a
  package manager upgrades a dependency's generated lockfile shape, not
  just its version pin). `update` changes *what's deployed*; `upgrade`
  changes *how it's deployed*.

**Tradeoffs:** Persisting manifests in this repo means every deploy/
update/upgrade is a real file change with a real diff to review before
applying — more friction than a silent, ephemeral apply, but it's the
friction Principle IV's GitOps requirement and this repo's own stated
purpose (holding real deployment configuration) explicitly call for, not
an accidental cost.
**Integration points:** the catalog write step (already decided in `002`/
`007`) happens after a successful `kubectl apply -k`, same as already
described in `architecture.md`'s deployment-pipeline summary — this doc
doesn't change that ordering, just fills in what happens immediately
before it.
**Data model implications:** none beyond the repo's own file layout
(`deployments/<app-name>/`) — no new database/service involved.

## Existing Codebase Evaluation

No `skipper` deployment code exists yet. `bbb-le`'s `deploy/k8s/`
(Kustomize base + overlays) remains the direct structural precedent,
simplified to one base per app with no overlay layer for v1, per the
domain assumption above.

## Security & Compliance

- Generated manifests never embed secret values directly — Kubernetes
  `Secret` objects reference Secret Manager-sourced values (already
  required by constitution Principle VI), and the persisted Kustomize
  base only ever contains a `Secret` *reference*, not its contents.
- Namespace-per-environment separation (already required by Principle
  IV) is unaffected by this doc's "no overlay layer for v1" decision —
  that's about *environments* (dev/staging/prod). The *per-app*
  namespace strategy this bullet originally asserted as "already
  decided" was not actually decided anywhere — corrected by
  [019-gke-namespace-strategy.md](019-gke-namespace-strategy.md): each
  app gets its own namespace (`skipper-<app-name>`), with the
  `Namespace` resource declared in that app's own Kustomize base
  alongside its Deployment/Service, per
  [019 → Decision Log](019-gke-namespace-strategy.md#decision-log).

## Performance, Reliability & Operations

- `kubectl apply -k` is idempotent by construction — re-running `update`/
  `upgrade` against an unchanged base is a no-op, consistent with the
  "safe to re-run" requirement already in the constitution's Security &
  Operational Posture section.
- Rollback: since manifests are git-committed, reverting a bad
  `update`/`upgrade` is a normal git revert + re-apply — no separate
  rollback tooling needed beyond what git + `kubectl apply -k` already
  provide.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| This repo accumulating many `deployments/<app-name>/` directories over time without a clear convention drifts into disorganization | Low–Medium | Low | One consistent generated structure per app, written by `skipper` itself rather than hand-edited ad hoc, keeps this self-consistent as the app count grows | mona |
| Conflating `update` and `upgrade` semantics in actual CLI usage, since the distinction (what's deployed vs. how) is subtle | Medium | Low–Medium | Document the distinction clearly in `skipper`'s own `--help` text once implemented; this doc's Decision Log is the source of truth for the intended meaning | mona |

## Open Questions

None — this doc's findings (Kustomize, one base per app, persisted/
committed manifests, the update/upgrade distinction) are all resolved
directly from existing constitutional requirements (Principle II's named
lifecycle, Principle IV's GitOps-friendly mandate, Principle VIII's
repo-scope framing) and the already-named `bbb-le` precedent — none of
this required an owner preference call beyond what's already decided
elsewhere.

## Decision Log

- **2026-06-24** — **Kustomize (via `kubectl apply -k`, no separate
  binary) is the manifest-generation/templating mechanism — not Helm,
  raw YAML, or a full IaC tool.** — Rationale: built into `kubectl`
  natively, no new dependency to install or version-pin; directly matches
  the already-named `bbb-le` precedent; Helm's release/chart machinery
  has no real audience for a single-owner, one-cluster deployment target
  (Principle I). — Decided by: mona.
- **2026-06-24** — **One Kustomize base per app, no environment-overlay
  layer for v1** (unlike `bbb-le`'s three overlays). — Rationale:
  `skipper` targets exactly one GKE cluster per platform instance;
  carrying over multi-environment overlay structure with no second
  environment to overlay against would be unjustified design weight
  (Principle I). Revisit only if a real second environment (e.g. staging)
  is actually introduced. — Decided by: mona.
- **2026-06-24** — **Generated manifests are persisted and committed in
  this repo** (`deployments/<app-name>/`), **not computed-and-applied
  ephemerally.** — Rationale: directly required by constitution Principle
  IV's "GitOps-friendly" mandate and Principle VIII's framing of this
  repo as the place backend deployment configuration lives; an ephemeral
  model would satisfy neither. — Decided by: mona.
- **2026-06-24** — **`app update` changes what's deployed (e.g. an image
  reference); `app upgrade` changes how it's deployed (re-rendering an
  app's base against `skipper`'s current manifest-generation template).**
  — Rationale: gives the two lifecycle verbs named in constitution
  Principle II distinct, non-overlapping meanings rather than treating
  them as synonyms. — Decided by: mona.

## Internal References

- [003-backend-application-deployment.md](003-backend-application-deployment.md) — the deployment pipeline this doc fills in the manifest-generation mechanism for
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — `bbb-le`'s Kustomize reference, first named here
- [009-markdown-stash-application.md](009-markdown-stash-application.md) — precedent for rejecting a reference repo's multi-target design weight (rook's `space-id`) when this platform's actual scale doesn't need it, reused for the overlay-layer decision above
- [019-gke-namespace-strategy.md](019-gke-namespace-strategy.md) — resolves the per-app namespace strategy this doc had asserted without an actual citation
- [021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md) — resolves the per-app IAM/Workload Identity binding mechanism, added to the same Kustomize base this doc establishes
- [architecture.md](architecture.md) — deployment-pipeline summary this doc fills in
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I, II, IV, VI, VIII, all directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Kubernetes docs — Declarative Management of Kubernetes Objects Using Kustomize | https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/ | 2026-06-24 | Confirms `kubectl` has had native Kustomize support (`kubectl apply -k`) since v1.14, with no separate binary required — the basis for selecting Kustomize as a zero-new-dependency choice. |

## Appendix

None yet.
