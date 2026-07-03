# Research: Backend application deployment — off-the-shelf and custom container images

**Status:** Complete — all open questions resolved; ready for feature-item creation
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

`specs/research/roadmap.md` §0 flagged "backend custom-application stack"
as an open item distinct from `skipper`'s own CLI implementation language.
The owner's actual requirement is narrower and more concrete than "pick a
language": `skipper` needs to **deploy arbitrary container images onto the
GKE backend**, from three different sources, and most of those images are
plain backend services with no user-facing interface at all — only some
explicitly claim a web app and/or a TUI (the named precedent is
`github.com/charmbracelet/soft-serve`).

Three image sources to support:

1. **Off-the-shelf containers from a public registry** (e.g. Docker Hub) —
   no build step; `skipper` deploys the image as published.
2. **Custom images built from a GitHub repo** — `skipper`'s own scaffolded
   apps, or any other repo the owner points it at; needs a build step.
3. **Custom images built from a local directory** — code not yet pushed
   anywhere; needs a build step without depending on a remote repo existing
   first.

And, orthogonally, a custom image MAY also be pulled from a public
registry directly if it's already built and published there — so "build
vs. no build" is the real fork, not "off-the-shelf vs. custom."

- **Goal:** One coherent `skipper` deployment path that handles all three
  sources, converges on the same declarative GKE manifest shape regardless
  of source, and feeds the [002](002-hybrid-tui-layer.md) app catalog only
  for apps that explicitly declare a web or TUI interface.
- **In scope:** Build mechanism for sources 2/3, registry strategy for
  source 1 (including Docker Hub rate-limit exposure), how an app declares
  "I have a web app" / "I have a TUI" / "I'm headless" to the catalog.
- **Out of scope:** The backend *application* language/framework
  conventions for apps `skipper` itself scaffolds from scratch (a separate,
  narrower question — most off-the-shelf and externally-sourced images are
  written in whatever language their own upstream chose, which `skipper`
  has no control over).
- **Success criteria:** A confirmed deployment pipeline shape and a
  confirmed catalog-interface-declaration model, recorded in this doc's
  Decision Log.
- **Non-goals:** Auto-detecting whether an arbitrary third-party image has
  a web/TUI interface — per the owner's framing, this is always an explicit
  claim, never inferred.

## Background & Context

- **Relevant prior work:** Constitution Principle IV (Kubernetes-native,
  declarative-first) already requires production-sensible defaults
  regardless of image source. Principle V (GCP managed-services alignment)
  already names Artifact Registry as the container-image home and requires
  infra-as-code. Principle VI (secure-by-default) already requires images
  to come from "trusted, scanned sources" and be pinned to digests/tags —
  directly relevant to off-the-shelf, third-party images that `skipper`
  itself didn't build.
- **`github.com/charmbracelet/soft-serve`** — already the named precedent
  in [002](002-hybrid-tui-layer.md) for category 4 (TUI-only Kubernetes
  apps); it's also the concrete example of an off-the-shelf-style container
  that explicitly claims a TUI interface, anchoring the "explicit claim,
  never inferred" requirement here.
- **Domain assumptions:** Most deployed backend apps will be headless
  (databases, queues, internal services, third-party tools) and simply
  won't appear in the app catalog at all — only apps that opt in with a
  declared interface do.

## Libraries & Stack

Candidate mechanisms by source type:

| Source | Build needed? | Confirmed mechanism | Notes |
| --- | --- | --- | --- |
| Public registry (Docker Hub, etc.) | No | **Direct pull** — no Artifact Registry mirror | Simpler setup, no extra GCP infra to define; accepts exposure to Docker Hub's own anonymous-pull rate limits as a tradeoff (see Risks). |
| GitHub repo | Yes | GitHub Actions building the image and pushing to Artifact Registry, authenticated via Workload Identity Federation (already required by Principle V) | Reuses the WIF-based CI/CD pattern already established; no new auth mechanism needed. |
| Local directory | Yes | **`gcloud builds submit` (Cloud Build)** — no local Docker daemon dependency | Consistent with Principle I (minimal local dependency footprint); slower per-iteration than a local `docker build` would be, accepted as the tradeoff for not requiring Docker installed locally. |

**Selected:** Direct Docker Hub pulls for off-the-shelf images; Cloud Build
(`gcloud builds submit`) for local-directory builds; GitHub Actions + WIF
for GitHub-repo builds. See Decision Log.
**Rejected (with reason):** An Artifact Registry remote-repository mirror
in front of Docker Hub — adds GCP infrastructure to define and maintain
that isn't justified yet for a single-developer platform; revisit if
Docker Hub rate limiting actually becomes a real, observed problem. Local
`docker build` + push for the local-directory path — would require Docker
installed locally just to deploy, which Cloud Build avoids.

## Architecture Patterns

- **One deployment pipeline, three entry points.** `skipper`'s deploy
  command surface forks only at the "get an image into a pullable state"
  step — `skipper app add --image <ref>` (source 1, or any
  already-built/published custom image), `skipper app add --github <repo>`
  (source 2), `skipper app add --source <local-dir>` (source 3). All three
  converge on the same next step: generate/apply a declarative GKE
  manifest (Deployment, Service, etc.) per Principle IV, with the image
  reference pointing at wherever it ended up (Artifact Registry, possibly
  via a Docker Hub remote-repository mirror). *(Filled in — see
  [010-gke-manifest-generation-and-upgrade.md → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log):
  the manifest mechanism is **Kustomize** via `kubectl apply -k` (no
  separate binary), one base per app under `deployments/<app-name>/`,
  **persisted and committed in this repo** — not computed-and-applied
  ephemerally — per Principle IV's GitOps-friendly mandate. `skipper app
  update --image <new-ref>` patches the persisted base's image reference
  and re-applies; `skipper app upgrade` re-renders an app's base against
  `skipper`'s current manifest-generation template and re-applies — the
  two verbs have distinct meanings, not synonyms.)*
- **Catalog interface declaration is explicit and editable after the
  fact, never auto-detected.** An app's catalog entry (per
  [002](002-hybrid-tui-layer.md)) carries an interface declaration —
  originally sketched here as `web:<port>` and/or `tui-ssh:<port>` —
  supplied by the owner either at `skipper app add` time or afterward via
  a separate `skipper app set-interface` command, since an image's actual
  interface may not be known/decided until after it's running. No
  interface declared means the app is deployed but never appears in the
  unified TUI/web front door — it's a headless backend service like any
  database or internal tool. *(Amended — see
  [007-tui-shell-dispatch-and-catalog.md → Decision Log](007-tui-shell-dispatch-and-catalog.md#decision-log):
  this sketch was later refined into the actual 5-type schema
  `skipper` implements — `ssh-passthrough` (`host`, `port`, `target`,
  routed through a shared Wishlist gateway), `embedded-view` (`viewID`),
  `cli-exec` (`binaryPath`), `web` (`url`), and `none`. The
  add-time-or-afterward editability and the "no interface = never
  appears" behavior described here are unchanged by that refinement.)*
- **Off-the-shelf images are never modified, only configured.** `skipper`
  deploys published images as-is (env vars, resource limits, probes,
  volumes per Principle IV) — it doesn't rebuild or patch third-party
  images. If a soft-serve-style image already claims a TUI interface
  itself, declaring that in the catalog entry is just describing what's
  already true about the image, not adding new capability to it.
- **Tradeoffs:** Pulling directly from Docker Hub (confirmed) accepts
  rate-limit exposure as a real but currently-unobserved risk, in exchange
  for not building/maintaining an Artifact Registry mirror that isn't
  justified yet. Cloud Build for local-dir builds (confirmed) accepts
  slower per-iteration builds in exchange for not requiring Docker
  installed locally.
- **Integration points:** This deployment pipeline is what populates the
  app catalog from [002](002-hybrid-tui-layer.md) — every successful
  `app add` (and any later `app set-interface`) with a declared interface
  is a catalog write.

### Source control topology: GitHub directly, or mirrored to a GCP source control service?

Deployment, upgrade, and maintenance for GitHub-repo-sourced apps run
through GitHub's own CI/CD (GitHub Actions), not a developer-machine
`skipper` invocation per deploy — confirmed context for this section.
The open question this note addresses: should the repo `skipper` builds
from be used straight from GitHub, or mirrored into a GCP-native source
control service (e.g. Cloud Source Repositories)?

**Straight from GitHub (no mirror):**
- *Pro:* Zero additional infrastructure — this is already the established
  path (GitHub Actions + Workload Identity Federation → Artifact Registry,
  per the Decision Log above). One source of truth; no second copy that
  can drift.
- *Pro:* Matches Principle I (minimal-dependency bias) — no infrastructure
  to justify that isn't already required.
- *Pro:* GitHub Actions' WIF integration (`google-github-actions/auth`) is
  mature and already the chosen mechanism regardless of this decision.
- *Con:* CI/CD availability depends on GitHub's own uptime — though this
  only blocks *shipping new changes* during a GitHub outage; already-
  deployed apps keep running on GKE unaffected.
- *Con:* No GCP-native backup copy of source independent of the GitHub
  account/repo — mitigated by ordinary git practices (local clones, etc.),
  not a gap unique to this decision.

**Mirrored to a GCP source control service (e.g. Cloud Source Repositories):**
- *Con (load-bearing):* **Google has been deprecating Cloud Source
  Repositories** — no new repository creation since mid-2024, with the
  service being wound down. Building a new mirroring step around a
  service in active sunset is a real, concrete risk, independent of any
  other tradeoff below. **Verify Cloud Source Repositories' current
  status before considering this path at all** — if it's fully retired by
  the time this is implemented, the option doesn't exist.
- *Con:* Adds a sync mechanism (a second push target, or a pull-mirror
  job) that must itself be built, secured, and kept from drifting —
  exactly the kind of infrastructure Principle I says must justify itself
  against a real need, not added "just in case."
- *Con:* Two places source code lives means two access-control surfaces to
  audit, working against Principle VI (secure-by-default, least
  privilege) rather than for it.
- *Pro (historical, now weak):* Tighter native Cloud Build trigger
  integration — but Cloud Build already supports GitHub repos directly
  (2nd-gen GitHub App integration), so this advantage has mostly eroded.
- *Pro (situational, not applicable here):* Data-residency/compliance
  requirements to keep source inside the GCP perimeter — not a real
  constraint for a single-developer private platform.

**Recommendation:** Build straight from GitHub, no mirror. Even setting
the deprecation risk aside, mirroring adds infrastructure and attack
surface for a benefit (a GCP-native backup copy) that doesn't serve a
concrete, demonstrated need on this platform — a direct Principle I call.

## Existing Codebase Evaluation

No `skipper` deployment code exists yet — greenfield. `bbb-le`'s
`deploy/k8s/` (Kustomize base + overlays) remains the closest existing
reference for the declarative-manifest shape this pipeline targets.

## Security & Compliance

- **Off-the-shelf, third-party images are an explicit supply-chain trust
  boundary.** `skipper` doesn't control what an upstream Docker Hub image
  contains. Per Principle VI, images must be pinned to specific digests
  (not floating `latest` tags) and come from sources the owner has
  deliberately chosen — `skipper` should make digest-pinning the default
  behavior of `app add --image`, not an opt-in flag.
- **Vulnerability scanning is informational, not a deploy gate.** Scan
  results (where available) are surfaced so the owner can make an informed
  call, but a deploy is never blocked on scan status — appropriate for a
  single-developer platform where the owner is the one deciding what to
  trust, rather than a policy enforced on their behalf. Revisit only if
  this stops being a single-developer-trust-model platform.
- **Custom images from GitHub repos** inherit the existing WIF-based CI/CD
  posture (Principle V/VI) — no new credential type introduced.
- **Local-directory builds** need their own credential story for pushing
  to Artifact Registry — likely the owner's own `gcloud` user credentials
  (interactive `gcloud auth login` session) rather than a service
  account, since this is an explicitly manual, local, single-developer
  flow, not CI.

## Performance, Reliability & Operations

- **Docker Hub rate limits** are a known, real operational risk for
  anonymous/unauthenticated pulls — accepted as a tradeoff for now (direct
  pulls confirmed); if this becomes a real, observed problem in practice,
  revisit with an Artifact Registry remote-repository mirror.
- **Local-dir build latency** (Cloud Build round-trip vs. local Docker) is
  a real, accepted iteration-speed cost of the confirmed Cloud Build
  decision — worth watching in practice; revisit only if it's actually
  disruptive to the dev workflow.
- Upgrade/version-pinning story for off-the-shelf images (e.g. bumping a
  mirrored Postgres image from one minor version to the next): **resolved
  by [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md)**
  — `skipper app update --image <new-ref>` patches the persisted
  Kustomize base's image reference and re-applies via `kubectl apply -k`.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Direct, unauthenticated Docker Hub pulls hit rate limits during real usage (e.g. pod restarts across a cluster) | Medium | Medium | Accepted for now (confirmed decision); add an Artifact Registry remote-repository mirror later if this is actually observed in practice | mona |
| An off-the-shelf image silently claims a TUI/web interface it doesn't actually have a stable, this is a config the developer wrote that just doesn't match the image's reality | Low | Low | This is explicitly the owner's responsibility (explicit claim, never inferred) — no enforcement needed beyond documenting the expectation clearly | mona |
| Floating tags (e.g. `:latest`) on off-the-shelf images cause silent, unreviewed behavior changes on redeploy | Medium | Medium | Default `app add --image` to require/resolve a digest, not just a tag | mona |
| Local-directory builds drift from what CI would have produced for the same source, if it's later pushed to GitHub | Low–Medium | Low–Medium | Treat local-dir builds as deliberately ephemeral/dev-only; the GitHub-repo path is the source of truth once a repo exists | mona |
| Mirroring source to Cloud Source Repositories turns out to be building on a service Google has deprecated/is winding down | Medium–High (if pursued) | Medium | Recommended against pursuing this path at all (see Source control topology note); verify current CSR status if ever reconsidered | mona |

## Open Questions

All questions originally listed here are resolved — see the Decision Log.

- [x] ~~Exact upgrade/version-pinning command shape for off-the-shelf
  images~~ — **Resolved by
  [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md):**
  `skipper app update --image <new-ref>` patches the persisted Kustomize
  base's image reference and re-applies via `kubectl apply -k` (see
  Performance, Reliability & Operations above).

## Decision Log

- **2026-06-24** — **Off-the-shelf images are pulled directly from Docker
  Hub (or whatever public registry), with no Artifact Registry mirror in
  front.** — Rationale: avoids building/maintaining GCP infrastructure
  (a remote-repository mirror) that isn't justified yet for a
  single-developer platform; Docker Hub rate-limit exposure is accepted as
  a tradeoff, revisited only if it's actually observed as a real problem.
  — Decided by: mona.
- **2026-06-24** — **Local-directory builds use `gcloud builds submit`
  (Cloud Build), not local `docker build`.** — Rationale: no local Docker
  daemon dependency, consistent with Principle I's minimal-local-footprint
  bias; slower per-iteration builds accepted as the tradeoff. GitHub-repo
  builds continue to use GitHub Actions + Workload Identity Federation
  (unchanged, already required by Principle V). — Decided by: mona.
- **2026-06-24** — **Vulnerability scanning is informational only — never
  a deploy-blocking gate.** Scan results are surfaced for awareness; the
  owner decides what to trust. — Rationale: appropriate for a
  single-developer, single-trust-model platform; revisit only if that
  changes. — Decided by: mona.
- **2026-06-24** — **Catalog interface declarations (web/TUI/none) are
  settable both at `skipper app add` time and afterward, via a separate
  command (e.g. `skipper app set-interface`).** — Rationale: an image's
  actual interface may not be known/decided until after it's running;
  locking the declaration to add-time only would force unnecessary
  remove-and-re-add cycles. — Decided by: mona.
- **2026-06-24** — **GitHub-repo-sourced apps build straight from GitHub
  — no mirroring to a GCP source control service (e.g. Cloud Source
  Repositories).** Deployment, upgrade, and maintenance run through
  GitHub's own CI/CD (GitHub Actions), not a developer-machine `skipper`
  invocation per deploy. — Rationale: zero added infrastructure, one
  source of truth, consistent with Principle I; Cloud Source Repositories'
  active deprecation (no new repos since mid-2024) makes building a new
  mirroring step around it a concrete risk independent of any other
  tradeoff; the historical "tighter Cloud Build integration" advantage of
  mirroring has eroded now that Cloud Build supports GitHub repos
  directly. See "Source control topology" under Architecture Patterns for
  the full pro/con analysis. — Decided by: mona.

> **Format note (2026-07-03):** this doc predated the current
> `research-template.md`; its single `## References` list has been split into
> `## Internal References` + `## Web Citations` to match the template. Note:
> several load-bearing external claims in the body — Cloud Source Repositories'
> deprecation ("no new repos since mid-2024"), Docker Hub anonymous-pull rate
> limits, and Cloud Build's 2nd-gen GitHub integration — were asserted without
> a recorded source; they should be cited if this doc is revisited.

## Internal References

- [architecture.md](architecture.md) — backend layer = GKE-hosted custom
  apps + GCP managed services
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — app catalog this
  deployment pipeline feeds; soft-serve as the TUI-claim precedent
- [roadmap.md §0](roadmap.md#0-outstanding-design-questions) — "backend
  custom-application stack" open item this doc narrows

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| `monamaret/bbb-le` | https://github.com/monamaret/bbb-le | 2026-06-24 | Closest existing reference for the declarative GKE manifest shape (Kustomize base + overlays) this pipeline targets. |
| `charmbracelet/soft-serve` | https://github.com/charmbracelet/soft-serve | 2026-06-24 | External precedent for an off-the-shelf-style image explicitly claiming a TUI interface (the "explicit claim, never inferred" anchor). |

## Appendix

None yet.
