# Research: Firebase Hosting + Cloud Run deployment mechanics

**Status:** Complete — all questions resolved directly from each tool's documentation and the platform's existing WIF precedent; ready to update specs/research/005-generic-app-scaffold.md and any future live-mode feature item
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[005-generic-app-scaffold.md](005-generic-app-scaffold.md) decided this
scaffold's web layer is GCP-native (Firebase Hosting for the static
shell, Cloud Run for live/dynamic paths) — but never specified *how*
`skipper` actually deploys to either service, leaving the same kind of
black box [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md)
resolved for GKE. [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md)
explicitly flagged this as the next gap. This doc mirrors `010`'s
structure for the web layer: deployment mechanism, persistence model, and
update semantics for both Firebase Hosting and Cloud Run.

- **Goal:** Confirm the deployment mechanism, persisted-config location,
  and update semantics for Firebase Hosting (static + rewrites) and
  Cloud Run (live mode), narrow enough to update `005` and any feature
  that eventually exercises live mode (chat's eventual web client, or a
  future live-mode generic-scaffold instance).
- **In scope:** Deployment tooling for Hosting and Cloud Run; how
  Hosting's rewrites wire to Cloud Run for live mode; persistence
  model (mirroring `010`'s GitOps-committed approach); CI/CD auth
  (must stay keyless, per the platform's existing WIF-only posture).
- **Out of scope:** Re-opening `005`'s GCP-native-vs-Cloudflare decision,
  or re-deciding GKE's own deployment mechanism (`010`, unaffected by
  this doc).
- **Success criteria:** A confirmed mechanism and persistence model,
  recorded in this doc's Decision Log.
- **Non-goals:** Designing any specific app's actual Hosting/Cloud Run
  config values (domains, scaling limits) — this doc is about the
  generation/deployment *mechanism*, not per-app tuning.

## Background & Context

- **Relevant prior work:**
  - [005 → Decision Log](005-generic-app-scaffold.md#decision-log) — GCP-native web hosting (Firebase Hosting + Cloud Run); static mode needs no Cloud Run at all; live mode routes specific paths to Cloud Run via Hosting rewrites.
  - [010 → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log) — the analogous GKE pattern this doc mirrors: a declarative, persisted, version-controlled manifest, applied via a CLI command, with distinct `update`/`upgrade` semantics.
  - [011 → Open Questions](011-shared-operation-definitions-and-mcp-sdk.md#open-questions) — explicitly named this gap as the next research item.
  - [003-backend-application-deployment.md → Decision Log](003-backend-application-deployment.md#decision-log) — GitHub Actions + Workload Identity Federation is the platform's existing keyless CI/CD pattern; this doc extends it to a new deploy target rather than introducing a second auth mechanism.
- **Domain assumption:** No app in this release actually exercises live
  mode yet (the dashboard, `005`'s v1 instance, is static-only) — this
  doc's findings apply to the *next* app that needs live mode (chat's
  eventual web client is the named candidate), not anything in the
  current release.
- **Stakeholders:** mona (sole owner/operator).

## Libraries & Stack

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **Firebase CLI** (`firebase deploy --only hosting`) | Deploy static assets + Hosting config (including Cloud Run rewrites) | Apache-2.0 | Mature, the canonical tool for this exact job | Actively maintained by Google | High | Confirmed via Firebase's own docs (see Web Citations): rewrites-to-Cloud-Run config is deployed through the Firebase CLI — no Terraform/`gcloud` alternative is documented for this specific step. |
| **Cloud Run declarative YAML** (`gcloud run services replace service.yaml`) | Deploy/update the Cloud Run service itself, for live mode | Apache-2.0 (`gcloud`) | Mature, documented first-class deployment path | Actively maintained by Google | High | Confirmed via Cloud Run's own docs (see Web Citations): a standard Knative-style `Service` manifest, version-controllable, applied via one `gcloud` command — directly parallel to Kustomize + `kubectl apply -k` from `010`. |
| `terraform-provider-google[-beta]`'s Firebase Hosting resources (`google_firebase_hosting_site`/`_version`/`_release`) | Alternative IaC-managed Hosting deploys | MPL-2.0 | Beta status as of this research (see Web Citations) | Actively maintained (Google + HashiCorp) | Rejected | Beta maturity for the exact resources needed, and Firebase's own documentation routes the rewrites-to-Cloud-Run workflow through the CLI, not Terraform — consistent with `010`'s reasoning against introducing a second declarative-config toolchain when a native, purpose-built one already exists. |
| `FirebaseExtended/action-hosting-deploy` (GitHub Action) | CI/CD deploy step for Hosting | Apache-2.0 | Official Firebase-maintained GitHub Action | Actively maintained | High | Confirmed via its own issue tracker (see Web Citations): supports a federated-credential file from `google-github-actions/auth`, i.e. **Workload Identity Federation**, no static service-account key — directly reuses the platform's existing keyless CI/CD posture. |

**Selected:** Firebase CLI for Hosting deploys (including rewrites
config); Cloud Run's own declarative YAML + `gcloud run services
replace` for the live-mode backend; `FirebaseExtended/action-hosting-
deploy` + `google-github-actions/auth` (WIF) for CI/CD — no new
credential type introduced.
**Rejected (with reason):** Terraform's Firebase Hosting resources — beta
maturity, and not the documented path for the rewrites-to-Cloud-Run
workflow this scaffold actually needs. See table.
**Open questions:** none for this section — resolved directly from each
tool's documented maturity/maintenance and the existing WIF precedent,
not an owner preference call.

## Architecture Patterns

- **Persisted, declarative config per app, mirroring `010`'s GKE
  pattern exactly.** Each web-layer app gets a directory (e.g.
  `deployments/<app-name>/web/`) holding: `firebase.json` (Hosting
  config, including any `rewrites` block for live mode), `.firebaserc`
  (project alias), and — **only for live-mode apps** — `service.yaml`
  (the Cloud Run Knative manifest). All committed and version-controlled
  in this repo, same GitOps rationale as `010`.
- **Static mode**: `firebase.json` has no `rewrites` block; `skipper`
  runs `firebase deploy --only hosting` against the persisted config and
  built static assets. No Cloud Run involvement at all — exactly `005`'s
  already-confirmed static-mode definition.
- **Live mode**: `firebase.json`'s `rewrites` array routes specific paths
  (e.g. `/api/**`) to the Cloud Run service by `serviceId`/`region`; the
  Cloud Run service itself is deployed separately via `gcloud run
  services replace deployments/<app-name>/web/service.yaml`, **before**
  the Hosting deploy that wires the rewrite to it (ordering matters: the
  rewrite target must exist before Hosting traffic can route to it).
- **`skipper app update --image <new-ref>`** (for a live-mode app):
  patches the persisted `service.yaml`'s container image and re-applies
  via `gcloud run services replace` — the direct web-layer analog of the
  GKE `app update` semantics from `010`. Hosting's own static content
  changes are a separate, simpler `firebase deploy --only hosting`
  re-run against updated build output — no equivalent "image reference"
  concept for Hosting itself.
- **`skipper app upgrade`**: as with `010`, re-renders an app's web-layer
  config (both `firebase.json` and, if present, `service.yaml`) against
  `skipper`'s current scaffold template and re-applies — distinct from
  `update`, same distinction `010` already established for GKE.

**Tradeoffs:** Running two separate deploy commands for live mode
(`gcloud run services replace` then `firebase deploy --only hosting`) is
more orchestration than a single unified command, in exchange for using
each service's own native, purpose-built deployment path rather than
forcing both through one tool that wasn't designed for the other (e.g.
trying to manage Cloud Run revisions through the Firebase CLI, which
isn't its job).
**Integration points:** the same GitHub Actions + WIF pattern already
used for GKE-bound builds (`003`) is reused here via
`FirebaseExtended/action-hosting-deploy` + `google-github-actions/auth`
— one CI/CD auth mechanism across both the backend (GKE) and web (Firebase/
Cloud Run) deployment targets, not two.
**Data model implications:** none beyond the repo's own file layout
(`deployments/<app-name>/web/`) — consistent with `010`'s "no new
service" finding for the analogous GKE question.

## Existing Codebase Evaluation

No `skipper` web-deployment code exists yet. biblio's own generated
GitHub Actions workflow (deploying to Cloudflare Workers) is a loose
structural precedent — "a generated CI/CD step deploys the web layer" —
but targets a different provider entirely, so only the *shape* (a
generated workflow file performing the deploy) transfers, not any
literal step.

## Security & Compliance

- CI/CD for both Hosting and Cloud Run deploys authenticates via
  Workload Identity Federation — confirmed available for
  `FirebaseExtended/action-hosting-deploy` (see Web Citations) — no
  static service-account key, consistent with constitution Principle VI
  and the existing GKE-deployment posture from `003`.
- Cloud Run service-to-service calls (e.g. to the GKE backend, for live
  mode's actual data) should use Workload Identity for the Cloud Run
  service's own runtime identity, mirroring the no-static-keys posture
  already required everywhere else on this platform — not a new
  exception. *(Mechanism filled in — see
  [021-iam-service-account-scoping-strategy.md → Decision Log](021-iam-service-account-scoping-strategy.md#decision-log):
  each app gets its own dedicated GCP service account
  (`skipper-<app-name>`); for Cloud Run specifically, that account's
  email is set directly via `spec.template.spec.serviceAccountName` in
  the `service.yaml` this doc already establishes — Cloud Run's own
  native field, no Config Connector or separate binding step needed.)*

## Performance, Reliability & Operations

- `gcloud run services replace` and `firebase deploy --only hosting` are
  both idempotent, versioned operations (Cloud Run revisions, Hosting
  releases) — re-running either against an unchanged manifest/build
  output is a safe no-op, consistent with the "safe to re-run" posture
  already required platform-wide.
- Rollback: both Cloud Run (revision history) and Firebase Hosting
  (release history) keep their own native rollback mechanisms
  independent of this doc's persisted-config model — `skipper` doesn't
  need to build custom rollback tooling, it can defer to each service's
  built-in history (Firebase Hosting's release list, Cloud Run's
  revision list).

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Deploying the Cloud Run service and the Hosting rewrite out of order (rewrite live before the Cloud Run service exists) causes a broken live-mode path | Low–Medium | Medium | Always deploy Cloud Run first, then Hosting — document this ordering explicitly in the generated CI/CD workflow, not left implicit | mona |
| GitHub Actions' OIDC token expiring mid-deploy on a long-running CI job (noted as a real, documented WIF gotcha) | Low | Low–Medium | Accepted risk for now — `skipper`'s own deploy jobs are expected to be short (a static build + one or two deploy commands); revisit only if real timeout failures are observed | mona |
| Two separate deploy tools (Firebase CLI, `gcloud`) for one logical "deploy this web app" operation adds orchestration surface vs. a single unified command | Medium | Low | Each tool is native/purpose-built for its own service — the alternative (forcing one tool to manage both) would be worse, not simpler | mona |

## Open Questions

None — this doc's findings (Firebase CLI for Hosting, Cloud Run's own
declarative YAML, persisted config under `deployments/<app-name>/web/`,
the `update`/`upgrade` distinction, and WIF-based CI/CD) are all resolved
directly from each tool's own documentation and the platform's existing
WIF precedent, mirroring `010`'s already-established pattern rather than
requiring a new owner preference call.

## Decision Log

- **2026-06-24** — **Firebase Hosting deploys (static assets + rewrites
  config) go through the Firebase CLI (`firebase deploy --only
  hosting`) — not Terraform.** — Rationale: the canonical, purpose-built
  tool for this exact job; Firebase's own documentation routes the
  rewrites-to-Cloud-Run workflow through the CLI specifically, with no
  documented Terraform/`gcloud` alternative; Terraform's Firebase Hosting
  resources are beta-status, consistent with `010`'s reasoning against
  introducing disproportionate IaC tooling when a native path already
  exists. — Decided by: mona.
- **2026-06-24** — **Cloud Run (live mode's backend) is deployed via its
  own declarative Knative-style YAML manifest, applied with `gcloud run
  services replace` — the direct web-layer analog of Kustomize +
  `kubectl apply -k` from `010`.** — Rationale: a standard, version-
  controllable manifest format with a first-class `gcloud` deployment
  path; keeps the same "persisted, declarative, CLI-applied" pattern
  consistent across both the GKE backend and the web layer's live mode.
  — Decided by: mona.
- **2026-06-24** — **Persisted config lives under
  `deployments/<app-name>/web/`** (`firebase.json`, `.firebaserc`, and
  `service.yaml` for live-mode apps), committed in this repo — mirroring
  `010`'s GKE persistence model exactly, for the same GitOps rationale.
  — Decided by: mona.
- **2026-06-24** — **CI/CD for both Hosting and Cloud Run deploys
  authenticates via Workload Identity Federation, using
  `FirebaseExtended/action-hosting-deploy` with `google-github-actions/
  auth`'s federated-credential output — no static service-account key.**
  — Rationale: confirmed supported by the official action; extends the
  platform's existing keyless CI/CD posture (`003`) to this new deploy
  target rather than introducing a second credential type. — Decided by:
  mona.
- **2026-06-24** — **`app update` patches a live-mode app's persisted
  `service.yaml` image and re-applies; `app upgrade` re-renders both
  `firebase.json` and `service.yaml` against `skipper`'s current
  template** — the same `update`-vs-`upgrade` distinction `010`
  established for GKE, applied consistently here. — Decided by: mona.

## Internal References

- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — the GCP-native web-hosting decision this doc fills in the deployment mechanism for
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) — the analogous prior doc this one mirrors in structure (manifest mechanism, persistence, update/upgrade semantics)
- [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) — the doc that flagged this gap as a follow-up
- [003-backend-application-deployment.md](003-backend-application-deployment.md) — the existing WIF-based CI/CD pattern this doc extends to a new deploy target
- [021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md) — the per-app service-account binding this doc's Cloud Run `serviceAccountName` field implements
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal-dependency bias) and Principle VI (secure-by-default), both directly load-bearing above

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Firebase — Serve dynamic content and host microservices with Cloud Run | https://firebase.google.com/docs/hosting/cloud-run | 2026-06-24 | Confirms the `firebase.json` `rewrites`-to-Cloud-Run config syntax and that the Firebase CLI (`firebase deploy`) is the documented deployment mechanism — no Terraform/`gcloud` alternative noted for this specific workflow. |
| Cloud Run — Deploying services | https://docs.cloud.google.com/run/docs/deploying | 2026-06-24 | Confirms `gcloud run services replace service.yaml` as a first-class declarative deployment path using a standard Knative-style `Service` manifest — the basis for treating Cloud Run's own deploy mechanism as parallel to Kustomize/`kubectl apply -k`. |
| `hashicorp/terraform-provider-google` — Firebase Hosting resources (Terraform Registry + GitHub issue #12955) | https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_hosting_site ; https://github.com/hashicorp/terraform-provider-google/issues/12955 | 2026-06-24 | Confirms Firebase Hosting Terraform resources exist but are beta-status as of this research — basis for rejecting Terraform as the primary mechanism. |
| `FirebaseExtended/action-hosting-deploy` — GitHub issue #174 | https://github.com/FirebaseExtended/action-hosting-deploy/issues/174 | 2026-06-24 | Confirms this official GitHub Action supports authenticating via a Workload Identity Federation credential file from `google-github-actions/auth`, rather than a static service-account key. |

## Appendix

None yet.
