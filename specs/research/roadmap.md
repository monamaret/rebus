# Roadmap: `skipper` implementation plan

**Status:** All §0 design questions are resolved across research docs 001–024
(004, the mobile device layer, remains **on hold** — stale premise). The six
first-release feature items enumerated in §1 are **fully designed in research
but not yet created on disk** — `specs/features/` currently holds only
`TEMPLATE.md` (the feature-item template, added 2026-07-03); none of the six
items exist yet, and creating them is the next step before Spec Kit runs
per-feature (see [IMPLEMENTATION-PLAN.md §5 Q6 and §6](../../IMPLEMENTATION-PLAN.md)).
Content-site-template work (001) remains **complete** but on hold per owner.
**Progress cursor:** → §1 (First major release) defines 6 feature items in
dependency order, **all designed in research but none yet created on disk**
(planned filenames under `specs/features/`): `001-tui-shell-and-styling.md`,
`002-markdown-reader-glow.md`, `003-off-the-shelf-image-deployment.md`
(generalized beyond Soft Serve), `004-markdown-stash-sync.md`,
`005-private-chat-1to1.md` (now **TUI-only for v1** — its web client was
deferred), `006-deployment-admin-dashboard.md` (the generic CLI+web+MCP
scaffold, proven via a deployment/admin dashboard — resolves the
admin-visibility question raised while refining 003). None started yet.
[001-web-layer-templates.md](001-web-layer-templates.md) (standalone-content-site
category) remains **complete** but on hold per owner.
[004-mobile-device-layer.md](004-mobile-device-layer.md) is **on hold** — its
premise was stale (no live web client exists in this release to pair mobile
against).

This roadmap is the single, current view of "what's next" and "where we
are" for `skipper`. Each section below corresponds to a feature item under
`specs/features/`, in dependency order, and is only added once the research
behind it has actually settled a buildable shape — nothing here should be
speculative scope.

## 0. Outstanding design questions

The single tracked location for open research/design items surfaced during
research but deliberately left open rather than guessed at.

- [x] ~~Web application layer stack specifics~~ — **Resolved 2026-06-24 by
  [001-web-layer-templates.md](001-web-layer-templates.md)** (standalone-
  content-site category) **and [005-generic-app-scaffold.md](005-generic-app-scaffold.md)**
  (backend-application-UI category). Final shape for 001: plain Markdown
  (no MDX, unchanged from biblio); Transmit, Commit, and Protocol all
  become **new sibling site types** (not upgrades to `log`/`docs`);
  Catalyst UI Kit stays scoped to the generic app scaffold, content sites
  keep shadcn/ui-on-Radix; content-site templates are **TypeScript-only**,
  no JS variant. See [001 → Decision Log](001-web-layer-templates.md#decision-log).
  Ready for `specs/features/` items.
- [x] ~~Concrete design of the three new content-site types~~ —
  **Resolved 2026-06-24 by [008-new-content-site-types.md](008-new-content-site-types.md).**
  Final shape: frontmatter schemas, route structure, and build pipeline
  for `broadcast`/`changelog`/`api-docs` — RSS via React Router v7
  **resource routes** (not Next.js route handlers); syntax highlighting
  via **Shiki**/`rehype-pretty-code` without MDX; search via **FlexSearch
  alone**, no Algolia; `broadcast` supports both self-hosted audio and
  external embeds. **Phased delivery: `changelog` first, then `api-docs`,
  then `broadcast`** — not one bundled feature. See
  [008 → Decision Log](008-new-content-site-types.md#decision-log).
  Ready for a `specs/features/` item (changelog first).
- [x] ~~Content-site scaffolding and deployment mechanics~~ — **Resolved
  2026-06-24 by [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md)**
  (corrected after an initial draft, see that doc's correction note).
  `010` and `012` each gave their deployment target the same treatment —
  this is the third and last (Cloudflare Workers, standalone content
  sites), but with a different shape: this category already has its own
  dedicated lifecycle tool. Final shape: **`skipper site new
  --type=<type> <repo-name>` wraps `biblio-cli new` directly**, generating
  a new, separate repo (Principle VIII); **every lifecycle action
  afterward — authoring, `index`, `update`, `build`, `deploy` — is
  `biblio-cli`'s job, not `skipper`'s**, including `biblio-cli deploy`
  running `wrangler deploy` via `biblio-cli`'s own generated CI workflow
  (accepting the Cloudflare API token that requires, since Cloudflare has
  no keyless alternative yet); no `skipper site update`/`deploy` command
  is built, since `biblio-cli` already provides one. Flags a real
  dependency: `biblio-cli` itself still needs to add support for the
  three new `008` site types before `skipper site new --type=changelog`
  etc. can work. See
  [017 → Decision Log](017-content-site-scaffold-and-deployment-boundary.md#decision-log).
  Ready to update [001](001-web-layer-templates.md) and
  [008](008-new-content-site-types.md).
- [x] ~~Hybrid TUI layer stack specifics~~ — **Resolved 2026-06-24 by
  [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md).** Final shape:
  - Categories 1 (dev-artifact/context) + 3 (GitHub/dev-toolchain
    integration) use a **stateless client** pattern: local Markdown stash
    (rook-style) for category 1; direct GitHub API calls (cached locally,
    no custom backend) for category 3.
  - Category 4 (TUI-only Kubernetes apps) uses the **SSH-session-served**
    pattern (bbb-le/Soft Serve): one SSH-served binary per app, with the
    app catalog (not a separate gateway) supplying the connect address.
    Category 2 (private chat)'s *general default* is the same
    SSH-session-served pattern with a simple custom protocol (no embedded
    IRC) — **but the first release's specific chat app is a scoped
    exception**, superseded 2026-06-24 to the stateless rook/sight pattern
    instead, because it needs both a TUI and a web client (impossible
    under SSH-session-multiplexing). See
    [002 → Decision Log](002-hybrid-tui-layer.md#decision-log).
  - App discovery (all categories) goes through the app catalog: a durable
    static artifact (GCS/Firestore) plus a rook-style client-side cache —
    no live registry service, no single point of failure for the common
    path.
  - See [002 → Decision Log](002-hybrid-tui-layer.md#decision-log) for the
    full rationale on each. Ready for a `specs/features/` item.
- [x] ~~Backend custom-application stack~~ — **Resolved 2026-06-24 by
  [003-backend-application-deployment.md](003-backend-application-deployment.md).**
  Final shape: one deploy pipeline (`skipper app add`) across three
  sources — off-the-shelf images pulled **directly from Docker Hub** (no
  Artifact Registry mirror), custom images from a **GitHub repo** (GitHub
  Actions + WIF, unchanged), and custom images from a **local directory**
  (`gcloud builds submit` / Cloud Build, no local Docker dependency).
  Vulnerability scanning is informational only, never deploy-blocking.
  Catalog interface declarations (web/TUI/none) are settable at `app add`
  time and editable afterward via `app set-interface`. GitHub-repo-sourced
  apps build **straight from GitHub, no mirror to a GCP source control
  service** — deployment/upgrade/maintenance run through GitHub Actions
  CI/CD, not developer-machine `skipper` invocations. See
  [003 → Decision Log](003-backend-application-deployment.md#decision-log).
  Ready for a `specs/features/` item.
- [x] ~~GKE manifest generation, persistence, and upgrade mechanics~~ —
  **Resolved 2026-06-24 by [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md).**
  Every backend feature referenced "a declarative GKE manifest per
  Principle IV defaults" as a black box — this fills it in. Final shape:
  **Kustomize** (via `kubectl apply -k`, no separate binary, matching the
  already-named `bbb-le` precedent) — not Helm, raw YAML, or a full IaC
  tool; **one base per app, no environment-overlay layer for v1** (unlike
  `bbb-le`'s three); **generated manifests are persisted and committed in
  this repo** (`deployments/<app-name>/`), not computed-and-applied
  ephemerally, per Principle IV's GitOps-friendly mandate; **`app update`
  changes what's deployed** (e.g. an image reference) **while `app
  upgrade` changes how it's deployed** (re-rendering against `skipper`'s
  current manifest template). All four resolved directly from existing
  constitutional requirements, no owner preference call needed. See
  [010 → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log).
  Ready to update `003` and every dependent feature item
  (003/004/005/006-feature) with this mechanism.
- [x] ~~GKE namespace strategy~~ — **Resolved 2026-06-24 by
  [019-gke-namespace-strategy.md](019-gke-namespace-strategy.md).** `010`
  asserted "per-app namespace isolation... already decided" with no
  actual prior decision behind it — this fills the gap. Final shape:
  **each app gets its own namespace** (`skipper-<app-name>`), not a
  single shared namespace — a bare `podSelector` `NetworkPolicy` only
  matches within its own namespace (confirmed via Kubernetes' own docs),
  making per-app namespaces the *simpler* way to satisfy Principle IV's
  least-access requirement, not a more complex alternative; the
  `Namespace` resource lives in each app's own Kustomize base, applied
  atomically with its Deployment/Service; shared infrastructure
  (Wishlist) gets its own namespace (`skipper-wishlist`) too; each
  namespace ships with a default-deny `NetworkPolicy` and explicit allow
  rules only for what that app needs. See
  [019 → Decision Log](019-gke-namespace-strategy.md#decision-log).
  Ready to update [010](010-gke-manifest-generation-and-upgrade.md).
- [x] ~~GCP IAM / Workload Identity service-account scoping strategy~~ —
  **Resolved 2026-06-24 by
  [021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md).**
  Principle V requires "least-privilege, per-service IAM bindings defined
  as code" but no doc ever named a mechanism — the IAM analog of `019`'s
  namespace gap. Final shape: **one dedicated GCP service account per
  app** (`skipper-<app-name>`), mirroring `019`'s per-app isolation
  exactly; **for GKE**, a Kubernetes `ServiceAccount` + Config
  Connector's `IAMServiceAccount`/`IAMPolicyMember` CRDs declare the
  Workload Identity binding in each app's own Kustomize base, applied via
  the same `kubectl apply -k` flow — no separate Terraform step; **for
  Cloud Run**, each app's `service.yaml` sets
  `spec.template.spec.serviceAccountName` directly — Cloud Run's own
  native field, no Config Connector needed there. Flags a real, unstarted
  dependency: Config Connector itself isn't confirmed installed on the
  target GKE cluster yet — cluster bootstrap has never been covered. See
  [021 → Decision Log](021-iam-service-account-scoping-strategy.md#decision-log).
  Ready to update [010](010-gke-manifest-generation-and-upgrade.md),
  [012](012-firebase-hosting-cloud-run-deployment.md), and
  [019](019-gke-namespace-strategy.md).
- [ ] **Mobile device layer** — **on hold**, per
  [004-mobile-device-layer.md](004-mobile-device-layer.md): its original
  trigger (chat's web client) is deferred, and the generic app scaffold's
  v1 instance defaults to static web (no live backend calls) — so there is
  currently no live web client anywhere in this release to pair a mobile
  client against. The WebAuthn-extends-to-mobile finding and the
  PWA-first recommendation remain useful defaults for whenever a live web
  client exists, but neither is blocking anything right now. Revisit once
  any app actually ships a live, authenticated web client.
- [x] ~~Private chat application — detailed design~~ — **Resolved
  2026-06-24 by [006-private-chat-application.md](006-private-chat-application.md).**
  Confirmed via discovery interview: transport+at-rest encryption only
  (not E2EE); indefinite retention; asymmetric admin/participant
  delete-edit permissions; opt-in local export; async-only delivery, no
  live mode; multiple separate 1:1 conversations supported; single-device
  identities for v1; manual-check-only sync (no background polling);
  in-app unread-state UI deferred to its own design phase; outside-domain
  participant uses a lighter standalone chat-only client, not the full
  TUI shell; no external compliance constraint. Confirmed via comparative
  research: **Cloud Firestore** storage with a
  `conversations/{conversationId}/messages/{messageId}` subcollection
  schema, incremental sync via an `updatedAt` field, clients never access
  Firestore directly (backend-RPC-mediated only); **one shared, public Go
  package** for wire types/RPC calls, imported by both the owner's TUI
  view and the guest's standalone client (adapted from sight's
  import-boundary pattern for a cross-repo split). **The backend is
  named `rebus` (`github.com/monamaret/rebus`) and the guest client is
  named `bateau` (`github.com/monamaret/bateau`), each in its own
  separate repository, not yet created** — the first concrete instance
  of Principle VIII's repo-separation requirement. See
  [006 → Decision Log](006-private-chat-application.md#decision-log).
  Ready to update [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md)'s
  Scope with these specifics.
- [x] ~~Generic web app + CLI scaffold~~ — **Resolved 2026-06-24 by
  [005-generic-app-scaffold.md](005-generic-app-scaffold.md).** Final
  shape: a reusable scaffold pairing a CLI (Cobra/Go) and a web app
  (static or live, configurable) as two thin clients over one stateless
  GKE backend, with optional MCP. **Web hosting is GCP-native (Firebase
  Hosting + Cloud Run), not Cloudflare** — a deliberate split from the
  standalone-content-site category covered by 001. MCP lives on the
  backend (not Cloud Run); CLI and MCP tools share one operation-
  definition set (the CLI does not render MCP App UI); MCP defaults to
  access-controlled, not public; the app registers in the catalog but the
  TUI shell launches it by shelling out to the CLI rather than embedding
  a separate view. The v1 example instance defaults to **static web + MCP
  on**, deferring live web mode (and its WebAuthn requirement) to later —
  e.g. when chat adopts this scaffold for its own deferred web client. See
  [005 → Decision Log](005-generic-app-scaffold.md#decision-log) for full
  rationale on each. The v1 instance's repo is named **`saratoga`** — see
  [024-dashboard-backend-repo-naming-and-boundary.md](024-dashboard-backend-repo-naming-and-boundary.md).
  Feature item created:
  [specs/features/006-deployment-admin-dashboard.md](../features/006-deployment-admin-dashboard.md).
- [x] ~~Shared CLI/MCP operation-definition mechanism~~ — **Resolved
  2026-06-24 by [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md).**
  `005` named "one operation-definition set" without ever specifying the
  mechanism — this fills it in. Final shape: **`modelcontextprotocol/
  go-sdk`** (the official MCP Go SDK, maintained with Google) — not
  `mark3labs/mcp-go` or `mcp-golang`; each operation is a **plain Go
  function `func(ctx, Input) (Output, error)`**, wrapped separately by an
  MCP adapter (`mcp.AddTool`) and a CLI adapter (a Cobra command) that
  both call the same function — no code generation; **auth is enforced
  per-adapter**, not centralized in the shared operation. See
  [011 → Decision Log](011-shared-operation-definitions-and-mcp-sdk.md#decision-log).
  Ready to update [R-005](005-generic-app-scaffold.md) and
  [specs/features/006-deployment-admin-dashboard.md](../features/006-deployment-admin-dashboard.md).
- [x] ~~WebAuthn implementation mechanics~~ — **Resolved 2026-06-24 by
  [020-webauthn-implementation-mechanics.md](020-webauthn-implementation-mechanics.md).**
  WebAuthn has been cited as "the decided mechanism" across `002`/`004`/
  `005`/`006`/`011` ever since `002`, but never given a library/schema —
  the MCP-adapter access-control check `011` enforces literally depends
  on this. Final shape: **`go-webauthn/webauthn`** (the actively
  maintained successor to `duo-labs/webauthn`, which is now superseded)
  — not a custom implementation (WebAuthn's protocol complexity is
  exactly the kind of security-critical logic Principle I's
  minimal-dependency bias doesn't apply to reimplementing); the existing
  owner-controlled identity registry implements the library's `User`
  interface directly (gaining a `webauthnCredentials` array field), with
  credential storage mirroring the library's own required fields
  (`id`, `publicKey`, `aaguid`, `signCount`, `cloneWarning` — the latter
  two **re-written on every login**, per the library's documented
  requirement). See
  [020 → Decision Log](020-webauthn-implementation-mechanics.md#decision-log).
  Ready to update [002](002-hybrid-tui-layer.md).
- [x] ~~Firebase Hosting + Cloud Run deployment mechanics~~ — **Resolved
  2026-06-24 by [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md)**
  — the web-layer analog of `010`'s GKE/Kustomize work, flagged as a
  follow-up by `011`. Final shape: **Firebase CLI** (`firebase deploy
  --only hosting`) for static assets + rewrites config — not Terraform
  (beta-status resources, no documented alternative for this workflow);
  **Cloud Run's own declarative Knative-style YAML**
  (`gcloud run services replace`) for the live-mode backend, directly
  parallel to Kustomize/`kubectl apply -k`; persisted config under
  `deployments/<app-name>/web/`, same GitOps rationale as `010`; CI/CD
  via `FirebaseExtended/action-hosting-deploy` + Workload Identity
  Federation — no static key, reusing the platform's existing keyless
  CI/CD posture. See
  [012 → Decision Log](012-firebase-hosting-cloud-run-deployment.md#decision-log).
  Not yet exercised by any v1 feature (the dashboard is static-only);
  applies to the next app that needs live mode (e.g. chat's eventual web
  client).
- [x] ~~CLI command-to-tool mapping and a standard app privilege/role
  convention~~ — **Resolved 2026-06-24 by
  [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md).**
  Consolidates every `skipper` command decided across `001`–`012` into
  one mapping table. Final shape for the privilege question: **`skipper`
  has no built-in role/permission system** — roles are entirely
  app-defined, gated by the platform's shared SSH-key identity registry,
  with role checks enforced at the per-adapter layer (`011`). **`rebus`'s
  admin operations (hard-delete, edit, invite) live within the owner's
  TUI-embedded chat view — not a new `skipper`-level command — and are
  never available in `bateau`, the guest's streamlined standalone
  client.** A generic `skipper app admin <app> <verb>` pass-through is
  explicitly deferred (only one app needs admin ops today). `rebus`'s
  Firestore schema gains an explicit `role` field per participant. See
  [013 → Decision Log](013-cli-command-mapping-and-privilege-model.md#decision-log).
  Flags a real gap: `skipper app remove`/`app list` don't exist yet, not
  designed in this doc. Ready to update
  [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md).
- [ ] **TUI shell — dispatch taxonomy and catalog schema gaps** —
  surfaced reviewing [001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)
  against 003/005/006: a third dispatch mechanism (CLI-exec, from 005)
  missing from 001's Scope; the catalog's 3-value interface schema (003)
  doesn't cover in-process views or CLI-exec; multi-screen apps (006's
  multiple chat conversations) unaddressed; theme propagation across
  dispatch boundaries and catalog refresh cadence both undecided.
  Confirmed so far: SSH-passthrough stays in v1 despite its narrowed
  (Soft-Serve-only) justification; **`github.com/charmbracelet/wishlist`
  becomes the shared network-multiplexing layer for Wish-pattern SSH
  apps** (one gateway behind one LoadBalancer, not one per app) — the
  catalog still discovers everything, SSH entries just resolve to the
  one Wishlist endpoint; amends but doesn't reverse
  [002](002-hybrid-tui-layer.md#decision-log)'s earlier rejection of a
  *custom-built* discovery gateway. Remaining open questions in
  [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md).
- [x] ~~Cross-app status-bar notifications, SSH-session notification
  visibility, non-manual sync~~ — **Resolved 2026-06-24 by
  [015-shell-notification-and-sync-freshness-model.md](015-shell-notification-and-sync-freshness-model.md).**
  All three follow-ups flagged during the 001/007 update: (1) a shared
  `shell.NotificationMsg` type, delivered via Bubble Tea's `Program.Send`,
  any embedded-view app can emit — not chat-specific; (2) **notifications
  queue and flush to the status bar only once the shell regains the
  terminal** — no real-time interrupt during SSH-passthrough/CLI-exec
  (owner decision: no mid-session interruption); (3) the platform-wide
  manual-only sync default is **unchanged**, but the shell exposes a
  per-view, opt-in `RefreshOnReentry` config flag (default `false`) so
  an individual app can trigger one sync on view re-entry without a
  timer or a global behavior change. See
  [015 → Decision Log](015-shell-notification-and-sync-freshness-model.md#decision-log).
  Ready to update [specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)
  and [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md).
- [x] ~~Notification/icon theming customization (emoji per app, per
  notification type, username-initial substitution)~~ — **Resolved
  2026-06-24 by [016-notification-and-theming-customization.md](016-notification-and-theming-customization.md).**
  Extends `015`'s `shell.NotificationMsg` with `Type` (an app-defined
  subtype, distinct from `Level`) and `Fields` (small structured data,
  e.g. a sender's username); extends `007`'s existing Lip-Gloss theme
  file with a new `notifications` section (icon per app/type, a
  configurable `usernameInitialEmoji` character map, cascading
  app+type → app default → global default resolution) — explicit
  config fields, no templating engine, kept deliberately light. Flags a
  known, real risk: `lipgloss` has documented open bugs in emoji-width
  calculation (cited GitHub issues), accepted as a cosmetic/upstream
  limitation, not something this config schema can fix. See
  [016 → Decision Log](016-notification-and-theming-customization.md#decision-log).
  Ready to update [specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)
  and [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md).
- [x] ~~Guest chat client (`bateau`): SSH session-state UX, SSH-as-auth
  re-check, full feature scope~~ — **Resolved 2026-06-24 by
  [014-guest-chat-client-scope-and-auth-ux.md](014-guest-chat-client-scope-and-auth-ux.md).**
  All three follow-ups flagged while reviewing chat (`006`): (1) auth
  resolution order — **SSH agent first, falling back to the private key
  file with a passphrase prompt if encrypted**, a missing key exits with
  a clear, actionable error; (2) **SSH-key auth remains correct for
  `bateau`** — the realistic guest profile makes key setup low-friction,
  and every alternative considered breaks the platform's
  one-identity-system principle or loses durable identity; (3) full
  feature scope — **GitHub Releases (`goreleaser`) + `go install`
  distribution from `github.com/monamaret/bateau`** (biblio's
  precedent), an interactive TUI plus `history`/`send` non-interactive
  commands (bbb-le's CLI shape), XDG config storage (rook-cli's
  convention). See
  [014 → Decision Log](014-guest-chat-client-scope-and-auth-ux.md#decision-log).
  Ready to update [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md).
- [x] ~~Stash's own detailed-design pass~~ — **Resolved 2026-06-24 by
  [009-markdown-stash-application.md](009-markdown-stash-application.md).**
  Final shape: **Cloud Storage for file content + Firestore for
  metadata** (Firestore's 1 MiB per-document cap rules out
  Firestore-only storage), the same `updatedAt`-incremental-sync pattern
  reused from chat (`006`), a flat (no multi-space) local cache mirroring
  rook-cli's `stash/` shape. **Soft delete** with a recovery window
  (not hard delete); **overwrite in place, no version history** on
  re-stash; **folder/tag organization** (`path`/`tags` metadata fields),
  not a flat list only. See
  [009 → Decision Log](009-markdown-stash-application.md#decision-log).
  Ready to update [specs/features/004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md)
  with these specifics.
- [x] ~~Stash backend repo naming and the skipper/implementation
  boundary~~ — **Resolved 2026-06-24 by
  [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md)**,
  applying the same repo-separation pattern already established for chat
  (`rebus`/`bateau`, `006`/`013`) to stash. Final shape: **the stash
  backend is named `pocket`** (`github.com/monamaret/pocket`, not yet
  created) — exactly **one** new repo (no second client/repo equivalent
  to `bateau`, since stash has no multi-party trust model); `skipper`'s
  own repo holds only the persisted Kustomize base
  (`deployments/pocket/`), catalog entry, and the owner's embedded-view
  adapter, which imports `pocket`'s published (public, not `internal/`)
  Go client package. See
  [018 → Decision Log](018-stash-backend-repo-naming-and-boundary.md#decision-log).
  Ready to update [009](009-markdown-stash-application.md) and
  [specs/features/004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md).
- [x] ~~App-catalog document contract (storage, top-level schema,
  versioning)~~ — **Resolved 2026-07-03 by
  [025-app-catalog-document-contract.md](025-app-catalog-document-contract.md).**
  `R-007` fixed the five per-*entry* interface types but never the container
  around them, and `R-002` left storage as "GCS or Firestore." Final shape:
  **a single JSON object (`catalog.json`) in Cloud Storage** (not Firestore —
  the catalog is small and read wholesale, so GCS's strong read-after-write/
  overwrite/list consistency + plain read-modify-write beat a document DB's
  unused query machinery); top-level `{ schemaVersion, generatedAt, entries[] }`
  wrapping `R-007`'s per-entry schema; integer `schemaVersion` bumped only on
  breaking changes; writes guarded by GCS generation preconditions; the object
  is read via the authenticated API (never public/CDN-cached). Unblocks
  `app set-interface` and all `kingfish` catalog reads. The bucket/object
  *name* stays deployment config (`IMPLEMENTATION-PLAN.md §5 Q2`). See
  [025 → Decision Log](025-app-catalog-document-contract.md#decision-log).
  Ready to inform `F-001`/`F-003` and the `kingfish` begin-state spec.
- [x] ~~Public client-package contract (`pocket`/`rebus` → `kingfish`/`bateau`)~~
  — **Resolved 2026-07-03 by
  [026-public-client-package-contract.md](026-public-client-package-contract.md).**
  Eight docs referenced "the client package" that importing repos consume at
  compile time, but none specified its API surface. Final shape: **one public
  (not `internal/`) Go package per backend** (cross-repo `internal/` imports
  are forbidden by the Go toolchain), containing the `Client`, wire types, and
  exported error sentinels, versioned by the backend module's own SemVer;
  standard surface `New(cfg) (*Client, error)` + ctx-first
  `func(ctx, In) (Out, error)` methods + a `sight`-shaped `Sync(ctx, since)`;
  auth via an injected `Authenticator`; errors matched with `errors.Is`, never
  string-parsed; breaking changes require a new major version. Standardizes the
  *shape* both packages follow; each backend's concrete method roster is
  finalized in its own feature item (`F-004` for `pocket`, `F-005` for
  `rebus`). See
  [026 → Decision Log](026-public-client-package-contract.md#decision-log).

## 1. First major release

Deliverable: the platform's first usable end-to-end slice — a styled TUI
shell, a local Markdown reader, a real off-the-shelf TUI-only app deployed
and reachable through the catalog, a custom artifact-sync app, a private
1:1 chat app (TUI-only for v1), and one instance of the generic CLI+web
scaffold. Confirmed and scoped 2026-06-24, directly from completed
research in [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md),
[003-backend-application-deployment.md](003-backend-application-deployment.md),
and [005-generic-app-scaffold.md](005-generic-app-scaffold.md).

**None of the six feature items below exist on disk yet** — `specs/features/`
holds only `TEMPLATE.md`. The `Planned feature spec` paths are the intended
filenames; the links resolve once the items are created from that template.

In dependency order:

- [ ] **TUI shell with configurable styling** — the launcher, app-catalog
  client (5-type schema, launch-only refresh + manual command), and
  **three** dispatch mechanisms (SSH-passthrough via a shared Wishlist
  gateway, in-process embedded view, CLI-exec — the first two sharing a
  common suspend-and-exec driver core), plus the theme system and a
  Glow-derived global UX convention (`Esc`/`q` back, `?` help drawer,
  status bar) every other feature below renders through or honors.
  Foundation for everything else in this release.
  **Planned feature spec (not yet created):** [specs/features/001-tui-shell-and-styling.md](../features/001-tui-shell-and-styling.md)
- [ ] **Local Markdown reader (Glow)** — local-only, no sync; the simplest
  instance of the stateless-client shape, plugged into the shell.
  **Planned feature spec (not yet created):** [specs/features/002-markdown-reader-glow.md](../features/002-markdown-reader-glow.md)
- [ ] **Off-the-shelf container image deployment** — the general
  capability to deploy any existing, already-built Docker image (not
  specific to one image), with an explicit TUI/web/none interface
  declaration. Soft Serve is the v1 test app (SSH-session-served, first
  end-to-end proof of the category-4 pattern); a second, unrelated
  headless image proves the "no interface" case. The platform-wide web
  admin/deployment view raised during refinement is now resolved — it's
  feature 006 below.
  **Planned feature spec (not yet created):** [specs/features/003-off-the-shelf-image-deployment.md](../features/003-off-the-shelf-image-deployment.md)
- [ ] **Markdown stash with local/remote sync (`stash` command)** — custom
  app (GitHub-repo build path), GCP-backed storage, rook-style local cache,
  sight-style incremental sync, listed in the TUI with last-updated
  timestamps.
  **Planned feature spec (not yet created):** [specs/features/004-markdown-stash-sync.md](../features/004-markdown-stash-sync.md)
- [ ] **Private 1:1 chat (TUI, v1 — web deferred)** — stateless
  rook/sight-pattern backend named **`rebus`** (`github.com/monamaret/rebus`,
  not yet created), Firestore-backed (`conversations/{id}/messages/{id}`
  subcollections, `updatedAt`-incremental sync), supporting multiple
  separate 1:1 conversations. Two v1 clients sharing one public Go client
  package published from `rebus`: the owner's TUI view (full admin
  capabilities — hard-delete, edit), living in `kingfish`'s own repo
  (the TUI shell's repo — see
  [022-tui-shell-repo-boundary.md](022-tui-shell-repo-boundary.md), which
  corrects this entry's earlier "living in `skipper`'s own repo"
  wording), and
  **`bateau`** (`github.com/monamaret/bateau`, not yet created), a
  separate, lighter standalone client for the outside-domain guest
  (send/read/hide only). Supersedes the general chat default for
  this specific app (see [002 → Decision Log](002-hybrid-tui-layer.md#decision-log)).
  Its originally-planned web client is deferred to the generic app
  scaffold below. Full detailed design in
  [006-private-chat-application.md](006-private-chat-application.md).
  **Planned feature spec (not yet created):** [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md)
- [ ] **Generic CLI+web app scaffold, proven via a deployment/admin
  dashboard** — scaffold capability (CLI+web+optional MCP, static or live
  web, GCP-native hosting) plus its concrete v1 instance: a deployment/
  admin dashboard showing app-catalog data + live GKE health (pod status,
  restarts, resource usage), static web + MCP on (unchanged 005 defaults),
  MCP on the backend, hosted on Firebase Hosting + Cloud Run. MCP tool
  responses include a plain-text/terminal-renderable visualization tier
  (ASCII charts/sparklines) so the same stats render in both the web
  dashboard and the TUI via the CLI. Surfaced while refining
  [003-off-the-shelf-image-deployment.md](../features/003-off-the-shelf-image-deployment.md).
  Backend, CLI, and static web client all live in one repo, named
  **`saratoga`** (`github.com/monamaret/saratoga`, not yet created) — see
  [024-dashboard-backend-repo-naming-and-boundary.md](024-dashboard-backend-repo-naming-and-boundary.md).
  **Planned feature spec (not yet created):** [specs/features/006-deployment-admin-dashboard.md](../features/006-deployment-admin-dashboard.md)
