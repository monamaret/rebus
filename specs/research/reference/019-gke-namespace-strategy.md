# Research: GKE namespace strategy — per-app isolation and naming convention

**Status:** Complete — per-app namespace strategy resolved directly from Kubernetes' own NetworkPolicy semantics, correcting an unsupported claim in 010; ready to update specs/research/010-gke-manifest-generation-and-upgrade.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[010-gke-manifest-generation-and-upgrade.md → Security & Compliance](010-gke-manifest-generation-and-upgrade.md#security--compliance)
asserts "the existing per-app namespace isolation, which stays as
already decided" — but no prior doc actually decided this. Constitution
[Principle IV](../../.specify/memory/constitution.md) only requires
"namespace-per-**environment** separation" and "network policies scoped
to least access" — it doesn't specify *how many namespaces per
environment* or *what isolates one app from another* within the single
GKE cluster `010` already confirmed this platform uses. This doc
resolves the actual gap `010` papered over: a concrete namespace
strategy.

- **Goal:** Confirm whether apps get their own GKE namespace or share
  one, the naming convention, and how this interacts with Principle IV's
  network-policy requirement — narrow enough to update `010` and every
  dependent feature item with the real mechanism.
- **In scope:** Namespace-per-app vs. shared-namespace; naming
  convention; where the `Namespace` resource itself is declared (each
  app's own Kustomize base vs. a separate bootstrap step); the resulting
  default `NetworkPolicy` shape.
- **Out of scope:** Re-opening `010`'s already-settled Kustomize/
  persistence/update-vs-upgrade decisions — this doc adds the missing
  namespace layer, it doesn't reconsider the rest.
- **Success criteria:** A confirmed namespace strategy, recorded in this
  doc's Decision Log.
- **Non-goals:** Designing the exact `NetworkPolicy` rules for any
  specific app (e.g. exactly which ports `rebus` needs open) — this doc
  establishes the namespace *boundary* the policies will be scoped
  against, not each app's own specific rule set.

## Background & Context

- **Relevant prior work:**
  - [.specify/memory/constitution.md Principle IV](../../.specify/memory/constitution.md) — "namespace-per-environment separation" and "network policies scoped to least access" — the two requirements this doc reconciles into one concrete mechanism.
  - [010 → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log) — one Kustomize base per app, no environment-overlay layer for v1 (a single GKE cluster, i.e. a single "environment") — the domain assumption this doc inherits.
  - [010 → Security & Compliance](010-gke-manifest-generation-and-upgrade.md#security--compliance) — the unsupported "per-app namespace isolation... already decided" claim this doc actually resolves.
- **Domain assumption:** Per `010`, this platform runs one GKE cluster
  (one "environment") for v1 — so "namespace-per-environment" alone would
  literally mean *one* namespace total, which gives `010`'s "network
  policies scoped to least access" nothing to scope against. This doc
  resolves that tension directly.
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

Not applicable — this is a Kubernetes-native namespacing decision, not a
library/service comparison. No new dependency is introduced.

## Architecture Patterns

- **One namespace per app, not one shared namespace for the whole
  cluster.** A bare `podSelector`-only `NetworkPolicy` only ever matches
  pods *within the same namespace* (confirmed via Kubernetes' own docs,
  see Web Citations) — if every app shared one namespace, "network
  policies scoped to least access" would require much finer-grained,
  harder-to-get-right pod-label-based rules to achieve the same
  isolation a namespace boundary gives for free. Per-app namespaces are
  therefore the *simpler* way to satisfy Principle IV's policy
  requirement, not a more complex alternative — directly consistent with
  Principle I once the actual tradeoff is examined, not assumed.
- **Naming convention: `skipper-<app-name>`** (e.g. `skipper-rebus`,
  `skipper-pocket`, `skipper-dashboard`), not bare app names. The
  `skipper-` prefix keeps every namespace this platform manages clearly
  identifiable in `kubectl get namespaces` output, and avoids any future
  collision if the same GKE cluster ever runs anything outside
  `skipper`'s own management (defensive, low-cost — one string prefix).
- **Shared platform infrastructure (the Wishlist gateway) gets its own
  namespace too** — e.g. `skipper-wishlist` — not bundled into any single
  app's namespace, since it's shared across every SSH-passthrough app,
  not owned by one.
- **The `Namespace` resource lives in each app's own Kustomize base**
  (`deployments/<app-name>/namespace.yaml`, listed in that base's
  `kustomization.yaml` resources), not a separate bootstrap step. `kubectl
  apply -k deployments/<app-name>/` then creates the namespace and its
  Deployment/Service together in one atomic apply — namespace creation
  is idempotent, so this is safe to re-run exactly like every other part
  of `010`'s already-established apply flow.
- **Default-deny `NetworkPolicy` per namespace, with explicit allow rules
  only for what an app actually needs** (e.g. the GKE-internal traffic to
  Firestore/Cloud Storage, or — for `rebus`/`pocket` — whatever ingress
  the backend's RPC API requires). This is the concrete mechanism behind
  Principle IV's "least access" phrase, now that there's a real namespace
  boundary to scope it against.

**Tradeoffs:** One small `Namespace` resource added to every app's
Kustomize base is negligible boilerplate (one YAML object, generated by
`skipper site new`/`skipper app add` automatically, not hand-written per
app) — the alternative (a shared namespace with finer-grained pod-label
policies) would actually be *more* YAML and harder to reason about, not
less, so this isn't a tradeoff against simplicity, it's aligned with it.
**Integration points:** Slots directly into `010`'s existing Kustomize-
base-per-app structure — the `Namespace` resource is just one more
resource in a base that already exists, not a new generation mechanism.
**Data model implications:** None — purely a Kubernetes-resource
concern, no new service or schema.

## Existing Codebase Evaluation

No `skipper` deployment code exists yet. `bbb-le`'s own Kustomize
structure (already the precedent `010` named for the base-per-app shape)
likely namespaces its own deployment similarly, though this wasn't
re-verified specifically for namespace strategy — the reasoning above is
derived from Kubernetes' own `NetworkPolicy` semantics directly, not a
re-confirmed bbb-le citation.

## Security & Compliance

- This doc is itself a security-relevant correction: it replaces an
  unsupported "already decided" claim in `010` with an actual, reasoned
  decision, directly serving Principle IV's least-access requirement
  rather than asserting compliance without a mechanism behind it.
- Per-app namespaces also bound the blast radius of an RBAC mistake —
  a `RoleBinding` scoped to one app's namespace can't accidentally grant
  access to another app's resources, unlike a single shared namespace
  where `Role`/`RoleBinding` scoping would need to rely entirely on
  resource-name or label selectors instead of a namespace boundary.

## Performance, Reliability & Operations

- No performance impact — namespaces are a free Kubernetes API
  organizational construct, not a resource with runtime cost.
- Rollback/update mechanics are unaffected — `kubectl apply -k` already
  handles the `Namespace` resource the same idempotent way it handles
  everything else in `010`'s already-established flow.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| A future app's manifest is generated without its `Namespace` resource (e.g. a template bug), silently landing in `default` instead | Low | Medium | The scaffold/deploy tooling should generate the `Namespace` resource as a fixed, non-optional part of every app's base template — not something an app can omit | mona |
| Cross-namespace traffic an app actually needs (e.g. `rebus` calling a shared platform service) is blocked by an overly strict default-deny policy with no allow rule added | Low–Medium | Medium | Each app's own `NetworkPolicy` allow rules are an implementation detail to get right per app at build time, not a flaw in the namespace-per-app strategy itself | mona |

## Open Questions

None — the per-app namespace decision is resolved directly from
Kubernetes' own documented `NetworkPolicy` semantics and Principle IV's
existing least-access requirement, not an owner preference call. The
naming convention (`skipper-<app-name>`) is a low-stakes, easily-revised
default, not something requiring owner sign-off beyond what's recorded
in the Decision Log below.

## Decision Log

- **2026-06-24** — **Each app gets its own GKE namespace
  (`skipper-<app-name>`), not a single shared namespace** — the
  `Namespace` resource lives in that app's own Kustomize base, applied
  atomically with its Deployment/Service. — Rationale: a bare
  `podSelector` `NetworkPolicy` only matches within its own namespace
  (confirmed via Kubernetes' own docs); per-app namespaces are the
  *simpler* way to satisfy Principle IV's "least access" requirement, not
  a more complex alternative. — Decided by: mona.
- **2026-06-24** — **Shared platform infrastructure (the Wishlist
  gateway) gets its own namespace** (`skipper-wishlist`), separate from
  any single app's namespace. — Decided by: mona.
- **2026-06-24** — **Each app's namespace ships with a default-deny
  `NetworkPolicy`, with explicit allow rules added only for what that
  app actually needs** — the concrete mechanism behind Principle IV's
  "least access" phrase. — Decided by: mona.

## Internal References

- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — the Kustomize-base-per-app structure this doc's `Namespace` resource is added to, and the source of the unsupported claim this doc resolves
- [021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md) — the directly analogous IAM/Workload Identity decision this doc's per-app isolation reasoning was reused for
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and Principle IV (Kubernetes-native, declarative-first — namespace-per-environment and least-access network policies), both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Kubernetes docs — Network Policies | https://kubernetes.io/docs/concepts/services-networking/network-policies/ | 2026-06-24 | Confirms `NetworkPolicy` is a namespaced resource, and that a bare `podSelector` rule only matches pods in the same namespace (cross-namespace traffic requires an explicit `namespaceSelector`) — the basis for this doc's "per-app namespaces are the simpler way to achieve least-access isolation" finding. |

## Appendix

None yet.
