# Research: GCP IAM / Workload Identity service-account scoping strategy

**Status:** Complete — scoping strategy and declaration mechanism resolved directly from Config Connector's and Cloud Run's own documented capabilities; one real dependency (Config Connector cluster installation) flagged, not yet started; ready to update specs/research/010-gke-manifest-generation-and-upgrade.md, 012-firebase-hosting-cloud-run-deployment.md, and 019-gke-namespace-strategy.md
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

Constitution [Principle V](../../.specify/memory/constitution.md)
requires "Workload Identity Federation for workload-to-GCP
authentication... and least-privilege, per-service IAM bindings defined
as code" — but no research doc has ever concretely resolved *how many*
service accounts exist, *which* app gets which one, or *where* the IAM
binding itself is declared. [012-firebase-hosting-cloud-run-deployment.md → Security & Compliance](012-firebase-hosting-cloud-run-deployment.md#security--compliance)
gestures at this ("Cloud Run service-to-service calls... should use
Workload Identity for the Cloud Run service's own runtime identity")
without naming a mechanism — the same shape of gap [019-gke-namespace-strategy.md](019-gke-namespace-strategy.md)
found and fixed for namespaces. This doc resolves the IAM/Workload
Identity equivalent, for both GKE (`010`/`019`) and Cloud Run (`012`).

- **Goal:** Confirm a per-app (or shared) service-account strategy, the
  concrete declaration mechanism for both GKE and Cloud Run, and a
  naming convention — narrow enough to update `010`, `012`, and `019`.
- **In scope:** GKE Workload Identity binding mechanism (declared as
  code, ideally via the same `kubectl apply -k` flow `010` already
  established); Cloud Run's own runtime-identity field; service-account
  naming convention and its real length constraint.
- **Out of scope:** Re-opening `010`/`012`/`019`'s already-settled
  decisions — this doc adds the missing IAM layer, it doesn't reconsider
  the rest.
- **Success criteria:** A confirmed scoping strategy and declaration
  mechanism, recorded in this doc's Decision Log.
- **Non-goals:** Enumerating each specific app's exact IAM role grants
  (e.g. precisely which Firestore/Cloud Storage roles `pocket` needs) —
  this doc establishes the *mechanism* and *one-SA-per-app* boundary,
  not each app's specific role list.

## Background & Context

- **Relevant prior work:**
  - [.specify/memory/constitution.md Principle V](../../.specify/memory/constitution.md) — Workload Identity Federation, no static keys, least-privilege per-service IAM bindings defined as code — the requirement this doc gives a concrete mechanism.
  - [019 → Decision Log](019-gke-namespace-strategy.md#decision-log) — the directly analogous per-app-isolation decision this doc mirrors, for IAM rather than network policy.
  - [010 → Architecture Patterns](010-gke-manifest-generation-and-upgrade.md#architecture-patterns) — the Kustomize-base-per-app structure this doc's GKE mechanism slots into.
  - [012 → Architecture Patterns](012-firebase-hosting-cloud-run-deployment.md#architecture-patterns) — the Cloud Run `service.yaml` this doc's Cloud Run mechanism extends.
- **Domain assumption:** Both GKE and Cloud Run workloads need to call
  GCP APIs (Firestore, Cloud Storage at minimum) without any static
  service-account JSON key, per Principle V — the only question is the
  concrete binding/declaration mechanism, not whether Workload Identity
  is used at all (already settled).
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **GCP Config Connector** (`IAMServiceAccount`/`IAMPolicyMember` CRDs) | Declare a GCP IAM service account and its Workload Identity binding as Kubernetes-native YAML | Apache-2.0 | Mature, official Google product | Actively maintained by Google | High | Confirmed via Config Connector's own docs (see Web Citations): these CRDs let the GCP-side IAM service account and its `roles/iam.workloadIdentityUser` binding be declared as plain YAML, applied via ordinary `kubectl apply` — slots directly into `010`'s existing Kustomize-base-per-app/`kubectl apply -k` flow, no second tool needed. |
| Terraform (`google_service_account`/`google_service_account_iam_member`) | Same purpose, via a separate IaC tool | MPL-2.0 | Mature | Actively maintained | Rejected (for this specific binding) | Already named in constitution Principle V as a *general* option for backing infrastructure — but introducing a second declarative tool/apply step (Terraform alongside `kubectl apply -k`) for something Config Connector already lets live in the *same* Kustomize base is unjustified complexity, per the same reasoning `010` used to pick Kustomize over a full IaC tool in the first place. |
| Manual, hand-run `gcloud iam service-accounts create` + binding commands | Same purpose, imperative | N/A | N/A | N/A | Rejected | Directly violates Principle IV's "declarative... not imperative" mandate and Principle V's "defined as code" requirement. |
| Cloud Run's native `serviceAccountName` field (already part of `012`'s `service.yaml`) | Bind a Cloud Run service to a specific GCP service account | N/A (built into the Knative spec) | Mature, documented core field | Google-maintained | High | Confirmed via Cloud Run's own YAML reference (see Web Citations): `spec.template.spec.serviceAccountName` takes the GCP service account's email directly — no Config Connector or separate binding step needed for Cloud Run specifically, since Cloud Run services run *as* the specified account natively. |

**Selected:** GCP Config Connector's `IAMServiceAccount`/`IAMPolicyMember`
CRDs for GKE workloads (added to each app's existing Kustomize base);
Cloud Run's native `serviceAccountName` field (added to each app's
existing `service.yaml`) for Cloud Run workloads — both declarative,
both reusing an already-established apply mechanism, no second tool.
**Rejected (with reason):** A separate Terraform-managed IAM layer
(duplicates what Config Connector already does inside the existing
Kustomize flow); imperative `gcloud` commands (violates Principle IV).
See table.
**Open questions:** none for this section — resolved directly from each
mechanism's own documented capability, not an owner preference call.

## Architecture Patterns

- **One dedicated GCP IAM service account per app, mirroring `019`'s
  per-app namespace decision exactly.** A shared service account across
  apps would mean a credential/permission compromise in one app
  potentially reaches every other app's GCP resources — the same
  reasoning `019` already applied to namespaces (least-access isolation)
  applies identically to IAM scope.
- **Naming convention: `skipper-<app-name>`**, consistent with `019`'s
  namespace convention — e.g. the GKE service account for `rebus` is
  `skipper-rebus@PROJECT_ID.iam.gserviceaccount.com`. GCP service account
  IDs are constrained to 6–30 characters, lowercase alphanumeric and
  dashes (confirmed via GCP's own docs, see Web Citations) — every app
  name in this platform's corpus so far (`rebus`, `bateau`, `pocket`,
  `dashboard`) comfortably fits with the `skipper-` prefix; a longer
  future app name would need a truncation/hash fallback, flagged in
  Risks rather than designed in detail here since no real case exists
  yet.
- **GKE mechanism**: each app's Kustomize base gains three more
  resources alongside the `Namespace`/`Deployment`/`Service` already
  established (`010`/`019`): a Kubernetes `ServiceAccount` (in that app's
  own namespace), a Config Connector `IAMServiceAccount` (the GCP-side
  account), and a Config Connector `IAMPolicyMember` granting
  `roles/iam.workloadIdentityUser` binding the two together — all applied
  via the same `kubectl apply -k deployments/<app-name>/` already
  established, no second apply step.
- **Cloud Run mechanism**: each app's `service.yaml` (per `012`) sets
  `spec.template.spec.serviceAccountName` directly to that app's
  dedicated GCP service account email — no Config Connector resource
  needed here, since Cloud Run's own native field handles the binding.

**Tradeoffs:** Three more YAML resources per GKE app's Kustomize base is
a small, fixed addition (generated automatically, not hand-authored per
app) — consistent with `019`'s own "negligible boilerplate" framing for
the analogous namespace resource.
**Integration points:** Directly extends `010`'s Kustomize-base
structure and `012`'s `service.yaml` structure — no new deployment
mechanism, just more resources in already-existing files.
**Data model implications:** None — purely GCP/Kubernetes IAM resources,
no application data model change.

## Existing Codebase Evaluation

No `skipper` deployment code exists yet, and Config Connector's own
installation status on the target GKE cluster is itself unconfirmed (see
Risks) — no reference repo (biblio, bbb-le, rook-reference, sight) uses
Config Connector, so this is a novel addition to this platform's stack,
not an adapted reference-repo pattern.

## Security & Compliance

- This doc is itself the concrete mechanism behind Principle V's
  "least-privilege, per-service IAM bindings defined as code" — replacing
  an unspecified requirement with an actual, reasoned design, the same
  role `019` played for Principle IV's namespace requirement.
- No static service-account JSON key is introduced anywhere in this
  design — Workload Identity (GKE) and Cloud Run's native runtime
  identity are both keyless by construction, consistent with the
  no-static-keys posture already required everywhere else.

## Performance, Reliability & Operations

- No performance impact — IAM bindings are a one-time reconciliation by
  Config Connector's controller, not a runtime cost on the request path.
- Config Connector's own reconciliation loop (detecting drift and
  re-applying desired state) gives this mechanism the same "safe to
  re-run" idempotency property already required platform-wide.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Config Connector isn't actually installed on the target GKE cluster yet — this doc assumes it's available, but cluster bootstrap itself has never been covered in this platform's research | Medium | Medium | Flagged as a real dependency, not designed here — cluster bootstrap (installing Config Connector, granting it its own elevated IAM permissions) needs its own treatment before any app's Kustomize base can rely on its CRDs | mona |
| A future app's name is long enough that `skipper-<app-name>` exceeds the 30-character service-account-ID limit | Low (no current app name is close) | Low | Not designed in detail here — a truncation or hash-suffix fallback would be a small, mechanical fix if and when a real app name actually triggers it | mona |

## Open Questions

None for the scoping strategy itself — resolved directly from Config
Connector's and Cloud Run's own documented capabilities. One real,
flagged dependency (Config Connector's installation status on the actual
cluster) is not a research question, just an unstarted prerequisite.

## Decision Log

- **2026-06-24** — **Each app gets its own dedicated GCP IAM service
  account (`skipper-<app-name>`), not a shared one** — mirroring `019`'s
  per-app namespace decision. — Rationale: bounds the blast radius of a
  credential/permission issue to one app, the same least-access reasoning
  already applied to namespaces. — Decided by: mona.
- **2026-06-24** — **For GKE: the Workload Identity binding (a
  Kubernetes `ServiceAccount`, a Config Connector `IAMServiceAccount`,
  and an `IAMPolicyMember` granting `roles/iam.workloadIdentityUser`) is
  declared in each app's own Kustomize base, applied via the existing
  `kubectl apply -k` flow** — not a separate Terraform step. — Rationale:
  Config Connector's CRDs let this live in the same declarative flow
  `010` already established; a second IaC tool would be unjustified
  complexity. — Decided by: mona.
- **2026-06-24** — **For Cloud Run: each app's `service.yaml` sets
  `spec.template.spec.serviceAccountName` directly to its dedicated GCP
  service account** — no Config Connector resource needed, since Cloud
  Run's native field already handles this. — Decided by: mona.

## Internal References

- [019-gke-namespace-strategy.md](019-gke-namespace-strategy.md) — the directly analogous per-app-isolation precedent this doc mirrors for IAM
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — the Kustomize-base structure this doc's GKE mechanism extends
- [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md) — the `service.yaml` structure this doc's Cloud Run mechanism extends
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle IV (declarative-first) and Principle V (Workload Identity Federation, least-privilege per-service IAM bindings defined as code — directly resolved by this doc), both load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Web search — Config Connector `IAMServiceAccount`/`IAMPolicyMember` Workload Identity binding | (search query, no single URL) | 2026-06-24 | Confirms Config Connector's CRDs let a GCP IAM service account and its Workload Identity binding (`roles/iam.workloadIdentityUser`, member format `serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA]`) be declared as plain YAML, applied via ordinary `kubectl apply` — the basis for this doc's GKE mechanism. |
| Cloud Run YAML reference — `serviceAccountName` | https://docs.cloud.google.com/run/docs/reference/yaml/v1 ; https://docs.cloud.google.com/run/docs/configuring/services/service-identity | 2026-06-24 | Confirms `spec.template.spec.serviceAccountName` is a native Knative/Cloud Run field taking the GCP service account email directly — basis for needing no separate binding step for Cloud Run. |
| Google Cloud IAM docs — service account ID constraints | (search query results, GCP IAM documentation) | 2026-06-24 | Confirms service account IDs are 6–30 characters, lowercase alphanumeric and dashes — basis for this doc's naming-convention length caveat. |

## Appendix

None yet.
