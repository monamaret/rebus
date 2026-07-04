# Research: CLI command-to-tool mapping, and a standard privilege/role convention for apps

**Status:** Complete — command mapping consolidated; privilege convention resolved directly from existing reference-repo precedent; chat's invite placement resolved; one real gap (`app remove`/`app list`) flagged for future work, not designed here
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

Across `001`–`012`, `skipper`'s own command surface and each underlying
tool it orchestrates (`kubectl`, `gcloud`, `firebase`, `ssh`/Wishlist,
GitHub Actions) have been decided piecemeal, doc by doc — there's no
single place that maps `skipper`'s commands to what they actually invoke.
Separately, chat (`006`) introduced an asymmetric admin/participant
privilege model (admin can hard-delete/edit/invite; participant can only
hide), but never specified *where* admin operations actually live — on
`skipper` itself, on a per-app CLI, or only inside an app's TUI view — nor
whether this is meant to generalize to other apps that might need
elevated-privilege operations later. This doc resolves both: a
consolidated command-mapping table, and a standard, reusable convention
for app-level privilege models.

- **Goal:** (1) One table mapping every `skipper` command decided so far
  to its underlying tool invocation(s) and the doc that decided it; (2) a
  standard convention for how apps express admin/elevated-privilege
  operations, resolving whether those operations belong on `skipper`
  itself or on the app; (3) apply that convention concretely to chat's
  admin-invite operation.
- **In scope:** Consolidating already-decided command mappings;
  designing the privilege-convention shape; chat's specific case.
- **Out of scope:** Designing new `skipper` commands that don't exist yet
  (e.g. `app remove`/`app list`) — flagged as gaps in Open Questions, not
  designed here. Re-opening any already-decided mechanism (Kustomize,
  Firebase CLI, the shared-operation pattern) — this doc organizes and
  extends those decisions, not reconsiders them.
- **Success criteria:** A confirmed command-mapping table and privilege
  convention, recorded in this doc's Decision Log, with chat's feature
  spec updated to reflect where its admin operations actually live.
- **Non-goals:** Building a generic, skipper-level permissions/RBAC
  system — see Architecture Patterns for why that's explicitly rejected.

## Background & Context

- **Relevant prior work:**
  - `010`, `012` — the GKE and web-layer deployment mechanisms (Kustomize/`kubectl`, Firebase CLI/`gcloud run`) this doc's command table consolidates.
  - `007` — the Wishlist-routed SSH-passthrough mechanism and the suspend-and-exec driver core, also part of the command table.
  - `006-private-chat-application.md → Decision Log` — the admin/participant asymmetry (hard-delete/edit/invite vs. hide-only) this doc's privilege convention generalizes.
  - `002-hybrid-tui-layer.md → Security & Compliance` — already names the precedent this doc's convention is built on: "Admin role is backend-only and separate from application access" (bbb-le's documented role split), and that key revocation/invite commands live on *each reference repo's own CLI* (`bbb admin invite`/`admin revoke-key` on bbb-le's own `bbb` binary; `rook-server-cli` as *rook's own*, separate admin tool) — **not** on a shared platform-level tool in any of the three reference repos.
  - the platform-level "orchestrate, don't reimplement" principle (Skipper Constitution v1.4.0 Principle II — "`skipper` orchestrates and wraps underlying tools... it MUST NOT reimplement their functionality"; inherited by rebus as platform context per the rebus constitution preamble, and distinct from rebus's own Principle II "Stateless RPC Backend") — directly extends to this doc's "skipper doesn't own app-level admin logic" conclusion below.
- **Domain assumption:** Chat is the only v1 app with an admin/participant
  distinction so far; the convention this doc proposes must hold up for
  *future* apps with similar needs without forcing chat's exact shape
  onto them.
- **Stakeholders:** mona (sole owner/operator, and the only "admin" that
  exists today for any app).

## Libraries & Stack

Not applicable in the usual sense — this doc organizes already-selected
tools (Kustomize, `kubectl`, Firebase CLI, `gcloud`, GitHub Actions, SSH/
Wishlist) rather than evaluating new candidates. No new library/service
decision is made here.

## Architecture Patterns

### Part 1 — Consolidated command-to-tool mapping

| `skipper` command | What it does | Underlying tool invocation(s) | Decided in |
| --- | --- | --- | --- |
| `skipper app add --image <ref>` | Deploy an off-the-shelf image | Generates a Kustomize base under `deployments/<app>/`, applies via `kubectl apply -k` | `003`, `010` |
| `skipper app add --github <repo>` | Deploy a custom image built from a GitHub repo | Triggers GitHub Actions (WIF-authenticated) to build + push to Artifact Registry, then the same Kustomize/`kubectl apply -k` step | `003`, `010` |
| `skipper app add --source <local-dir>` | Deploy a custom image built from local source | `gcloud builds submit` (Cloud Build) → Artifact Registry, then the same Kustomize/`kubectl apply -k` step | `003`, `010` |
| `skipper app add --github <repo>` *(web-layer instance, via the generic scaffold)* | Deploy a CLI+web(+MCP) app's web layer | `firebase deploy --only hosting` (static); additionally `gcloud run services replace` for the Cloud Run service if `web_mode: live` | `005`, `012` |
| `skipper app set-interface` | Declare/edit an app's catalog entry (`ssh-passthrough`/`embedded-view`/`cli-exec`/`web`/`none`) | Pure catalog metadata write (GCS/Firestore) — no underlying CLI tool invoked | `003`, `007` |
| `skipper app update --image <new-ref>` | Bump a deployed app's image | Patches the persisted Kustomize base (or Cloud Run `service.yaml` for web-layer live-mode apps) and re-applies (`kubectl apply -k` / `gcloud run services replace`) | `010`, `012` |
| `skipper app upgrade` | Re-render an app against `skipper`'s current manifest template | Re-renders + re-applies via the same tool as `update`, but changes *how* it's deployed, not *what's* deployed | `010`, `012` |
| *(implicit, part of `app add --image` for SSH-passthrough apps)* Wishlist gateway deploy | Stand up the shared SSH gateway | Its own Kustomize base/`kubectl apply -k`, plus a config write registering the new app's `target` in Wishlist's own config | `007` (amends `002`; plugs into `app add --image` from `003`/`010`) |
| `kingfish` *(no subcommand)* | Launch the TUI shell | Dispatches per catalog entry via one of: `ssh` (through Wishlist, for `ssh-passthrough`), `exec` of a local binary (`cli-exec`), or in-process view resolution (`embedded-view`) | `002`, `007`, `022` |
| `skipper site new --type=<type> <repo-name>` | Scaffold a new standalone content site in its own repo | Thin wrapper around `biblio-cli new <repo-name> --type=<type>`; every later lifecycle action (`index`/`update`/`build`/`deploy`) is `biblio-cli`'s own, not `skipper`'s | `017` |
| `skipper tui new <repo-name>` | Scaffold the TUI shell (`kingfish`) repo | Generator carrying the `kingfish` template (no pre-existing tool to wrap, unlike `site new`) — emits a new, separate repo scaffold | `022` |

This table is the authoritative cross-reference — individual docs above
remain the source of *rationale*, this table is the source of *lookup*.
*(Updated 2026-07-03: extended beyond this doc's original `001`–`012` scope to
add `skipper site new` (`017`) and `skipper tui new` (`022`), decided after
this doc was written. Two commands remain **decided-necessary but not yet
designed** — `skipper app remove` and `skipper app list` (see Open Questions
below) — and are deliberately absent from the rows above until a design pass
exists; this note keeps the table honestly authoritative rather than silently
incomplete.)*
*(Corrected 2026-06-24, see
[022-tui-shell-repo-boundary.md → Decision Log](reference/022-tui-shell-repo-boundary.md#decision-log):
the row above originally read `skipper` *(no subcommand)*, implying the
shell ran in-process inside `skipper`'s own binary. The shell — and the
catalog-dispatch logic this row describes — lives in `kingfish`, a
separate binary/repo from `skipper`; `skipper` has no shell-launching
behavior of its own.)*

### Part 2 — Standard privilege/role convention for apps

**Decided shape:**

- **`skipper` has no built-in concept of roles, admin, or permissions.**
  Per the platform "orchestrate, don't reimplement" principle (Skipper
  Constitution v1.4.0 Principle II — not rebus's own Principle II, which
  is the rebus-specific "Stateless RPC Backend"),
  `skipper`'s job ends at deploying, discovering, and dispatching to an
  app — what an app does with its own users/roles once reachable is the
  app's own domain logic, exactly as none of the three reference repos
  (bbb-le, rook-reference, sight) push their own role models up into a
  shared platform tool. Building a generic RBAC system into `skipper`
  itself, before a second app with a genuinely different role shape
  exists, is exactly the speculative abstraction Principle I rejects.
- **The *identity* mechanism is shared and standardized; the *role*
  semantics are not.** Every app's privileged operations are gated by
  the same SSH-public-key-based registry already used platform-wide (the
  owner-controlled registry from `002`) — an app's "admin" is simply a
  registered key flagged with elevated privilege *for that app
  specifically*, stored alongside that app's own participant records
  (e.g. chat's `conversations` schema gains a `role` field per
  participant, rather than a new, separate admin-identity system). This
  is the one thing standardized across apps: *how* identity is proven
  (SSH-key-signed challenge, no IdP), not *what* roles mean. (Chat's
  backend is named `rebus`, its guest client `bateau` — see
  [006 → Decision Log](006-private-chat-application.md#decision-log) —
  each in its own separate repo; the `role` field lives in `rebus`'s own
  Firestore schema, not anything `skipper` stores.)
- **Admin-only operations follow the same shared-operation-definition
  pattern already decided in `011`, with the role check enforced at the
  adapter layer.** `011` already decided auth is enforced per-adapter,
  not centralized in the shared `func(ctx, Input) (Output, error)`
  operation; this doc extends that same per-adapter enforcement to
  *role*, not just identity — an admin-only operation's MCP/CLI adapter
  checks the caller's role before invoking the shared function, exactly
  where it already checks auth.
- **Admin operations surface wherever the app's own interface already
  is — never as a new `skipper`-level command.** *(Note added
  2026-06-24, see
  [022-tui-shell-repo-boundary.md → Decision Log](reference/022-tui-shell-repo-boundary.md#decision-log):
  the owner's TUI-embedded chat view referenced below lives in
  `kingfish`'s own repo, a separate repo from `skipper` — not in
  `skipper` itself, which holds no shell-side code.)* For chat specifically
  (resolving the prompt's direct question): the owner's TUI-embedded
  chat view ([005-private-chat-1to1.md](../features/005-private-chat-1to1.md))
  is where admin operations (hard-delete, edit, **invite**) live for v1
  — a screen/action within that already-existing view, not a new
  `skipper`-level command, and **not** available in `bateau`, the
  guest's streamlined standalone client, which has no admin role by
  design (per `006`'s already-confirmed send/read/hide-only scope for
  participants). If `rebus` ever adopts the generic CLI+web scaffold for
  its deferred web client, its admin operations would *also* gain a CLI
  surface at that point (`rebus admin invite <pubkey>`, mirroring
  bbb-le's `bbb admin invite` naming, and living in `rebus`'s own repo,
  not `skipper`'s) via that scaffold's shared-operation mechanism — but
  that's a consequence of `rebus` adopting the scaffold, not a reason to
  build a CLI surface for admin operations now.
- **A generic `skipper app admin <app> <verb>` pass-through command is
  explicitly deferred, not built now.** This would be a thin,
  skipper-level convenience that forwards to whatever an app's own admin
  API defines, without `skipper` owning any role semantics itself — a
  real, legitimate idea, but with only one app (chat) needing admin
  operations today, there's no second data point to design a *generic*
  pass-through against yet. Revisit once a second app demonstrates the
  same need, mirroring this project's existing escalation pattern
  (deferred until a real, observed need, not built speculatively).

**Tradeoffs:** Keeping roles entirely app-defined means every app with an
admin/participant distinction re-derives its own role field and
enforcement — slightly more repetition than a shared role-management
library would have, in exchange for not committing to a generic
permissions abstraction before a second real use case exists to validate
its shape.
**Integration points:** this convention slots directly into `011`'s
already-decided per-adapter auth enforcement — no new enforcement layer,
just an additional check (role, not just identity) at the same point.
**Data model implications:** `rebus`'s Firestore schema
(`006-private-chat-application.md`) already implicitly has this — the
admin/participant identities in `conversations/{conversationId}` just
need an explicit `role` field per participant rather than relying on
"the conversation creator is always admin," to be robust if a
conversation's participants ever change. This schema lives in `rebus`'s
own repo, not `skipper`'s.

## Existing Codebase Evaluation

No `skipper` admin-tooling code exists yet. bbb-le's `bbb admin
invite`/`admin revoke-key` (on bbb-le's own CLI, not a shared tool) and
rook-server-cli (rook's *own*, separate admin binary) are the direct
precedents for "admin operations live on the app, not the platform,"
already cited in `002`.

## Security & Compliance

- No change to the already-decided security posture — admin operations
  still gate on the same SSH-public-key registry, no IdP dependency, no
  new credential type.
- Centralizing *no* role logic in `skipper` itself means a compromised
  `skipper` binary/config has no special "admin override" capability
  across apps — each app's own admin check is independent, which is a
  modest defense-in-depth benefit of not building the generic
  pass-through command, beyond just avoiding premature abstraction.

## Performance, Reliability & Operations

Not applicable — this doc is about command/role organization, not a new
runtime characteristic.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| A future app re-derives its own ad hoc role convention that drifts from this doc's pattern (shared identity, per-adapter role check) | Medium | Low–Medium | This doc is the recorded convention to point back to when scaffolding a new app's admin operations — not enforced by tooling, but documented clearly enough to follow | mona |
| Chat's Firestore schema implicitly assumes "creator = admin" without an explicit `role` field, breaking if participants/roles ever need to change | Low (1:1, fixed roles for now) | Low | Add the explicit `role` field now, while the schema is still unbuilt, rather than retrofitting later | mona |

## Open Questions

- [ ] **`skipper app remove`/`app list` don't exist yet** — every
  command decided so far adds or modifies an app; there's no documented
  way to remove one or list what's currently deployed (beyond the
  catalog, which only shows interfaced apps, not headless ones). Not
  designed in this doc — flagged as a real gap for a future pass. —
  owner: mona
- [ ] **Exact UI for chat's invite action within the TUI-embedded view**
  (a dedicated screen vs. a command-palette-style action) — an
  implementation detail for chat's own design, not a research blocker
  for this doc's convention. — owner: mona

## Decision Log

- **2026-06-24** — **`skipper` has no built-in role/permission system;
  app-level roles (admin, participant, etc.) are entirely app-defined,
  gated by the platform's already-shared SSH-key identity registry, with
  role checks enforced at the per-adapter layer already established in
  `011`.** — Rationale: matches all three reference repos' own pattern
  (admin tooling lives on each app's own CLI, never a shared platform
  tool); avoids a generic RBAC abstraction with only one real use case
  (chat) to design against. — Decided by: mona.
- **2026-06-24** — **Chat's admin operations (hard-delete, edit, invite)
  live within the owner's TUI-embedded chat view for v1 — not as a new
  `skipper`-level command, and not available in the guest's streamlined
  standalone client.** — Rationale: the owner is already inside that
  view for every other admin action; the guest app has no admin role by
  design, per `006`'s already-confirmed scope. — Decided by: mona.
- **2026-06-24** — **A generic `skipper app admin <app> <verb>`
  pass-through command is explicitly deferred**, not built now. —
  Rationale: only one app (chat) currently needs admin operations — not
  enough data to design a generic pass-through against; revisit once a
  second app demonstrates the same need. — Decided by: mona.
- **2026-06-24** — **Chat's Firestore schema gains an explicit `role`
  field per participant**, rather than implicitly treating "the
  conversation creator" as always-admin. — Rationale: robust against any
  future change to who holds the admin role in a given conversation,
  decided now while the schema is still unbuilt. — Decided by: mona.

## Internal References

- [002-hybrid-tui-layer.md → Security & Compliance](reference/002-hybrid-tui-layer.md#security--compliance) — the bbb-le/rook-server-cli precedent this doc's "admin lives on the app, not the platform" conclusion is built on
- [003-backend-application-deployment.md](reference/003-backend-application-deployment.md), [010-gke-manifest-generation-and-upgrade.md](reference/010-gke-manifest-generation-and-upgrade.md), [012-firebase-hosting-cloud-run-deployment.md](reference/012-firebase-hosting-cloud-run-deployment.md), [007-tui-shell-dispatch-and-catalog.md](reference/007-tui-shell-dispatch-and-catalog.md) — the command/tool decisions consolidated into Part 1's table
- [006-private-chat-application.md](006-private-chat-application.md) — the admin/participant asymmetry this doc generalizes into a standard convention
- [011-shared-operation-definitions-and-mcp-sdk.md](reference/011-shared-operation-definitions-and-mcp-sdk.md) — the per-adapter auth-enforcement pattern this doc's role-check extends
- [specs/features/005-private-chat-1to1.md](../features/005-private-chat-1to1.md) — the feature item this research will refine with the explicit `role` field and the invite-action placement
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — rebus Principle I (minimal-dependency bias); plus the platform-level "orchestrate, don't reimplement" principle (Skipper Constitution v1.4.0 Principle II, inherited as platform context per the rebus constitution preamble — rebus's own Principle II is "Stateless RPC Backend," a different principle). Both directly load-bearing above.

## Web Citations

No new external research was needed — this doc consolidates and extends
already-decided internal mechanisms and reference-repo precedents
already cited (and sourced) in `002`, `003`, `007`, `010`, `011`, `012`.

## Appendix

### Verification audit — 2026-07-03

Verification pass (deep-research plan, Note 2). This doc declares "No new
external research was needed," so verification was an internal-consistency
cross-check of every command-table row and every privilege-convention
citation against the frozen `reference/` docs it cites. All Decision Log
entries preserved; two command-table citation errors corrected; one
re-scope artifact in the constitution citations re-anchored.

**Command table (Part 1) — every row cross-checked against its `Decided in`
docs:**
- Rows 1–3 (`app add --image`/`--github`/`--source`), row 6 (`update
  --image`), row 7 (`upgrade`), row 10 (`site new`), row 11 (`tui new`)
  all verify cleanly — the cited reference docs (003, 005, 007, 010, 012,
  017, 022) decide exactly what each row claims.
- **Row 8 (Wishlist gateway deploy) — citation corrected.** The
  Wishlist-as-shared-gateway decision is made entirely in `007` (which
  amended `002`'s earlier rejection); `003` never mentions Wishlist. The
  row's `Decided in` cell now reads `007` (amends `002`), with the
  `app add --image` scaffolding it plugs into traced to `003`/`010`.
- **Row 9 (`kingfish` dispatch) — citation corrected.** It cited `001`,
  but reference doc `001` is the web-layer-templates doc, not the TUI
  shell; the shell's interaction-model/dispatch research is `002`. The
  cell now reads `002`, `007`, `022`.
- Rows 4 (web-layer `app add --github`) and 5 (`set-interface`) verify,
  with a noted minor imprecision left as-is: the `--github` coupling is
  transitive via `003`, and the GCS/Firestore catalog-storage provenance
  traces to `002` (not `007`, which inherits it). Defensible; not changed.

**Open Questions confirmed still open:**
- `skipper app remove` / `skipper app list` — grep across all of
  `specs/research/reference/` confirms neither command is designed or
  decided anywhere; they remain a real gap, exactly as 013's inline note
  and Open Question state.
- Invite-action UI within the TUI-embedded view — still an implementation
  detail for chat's own design, not resolved here.

**Privilege convention (Part 2) citations verified:**
- `002` (bbb-le `admin invite`/`admin revoke-key`, rook-server-cli
  `user register-key` precedent) — present and correctly applied.
- `011` (auth "enforced per-adapter, not centralized in the shared
  operation") — present verbatim; 013's role-check correctly extends it.

**Constitution citations re-anchored (re-scope artifact):**
- This doc was written in `skipper` against the Skipper Constitution, then
  re-scoped into the rebus repo. Its three references to "Principle II
  ('orchestrate, don't reimplement')" dangled against the rebus
  constitution actually at `.specify/memory/constitution.md`, where
  Principle II is "Stateless RPC Backend" — a different principle. The
  "orchestrate, don't reimplement" concept is the Skipper Constitution
  v1.4.0's Principle II (per the rebus constitution's own preamble, which
  records that derivation). All three references (Background, Part 2
  prose, Internal References) are re-anchored to identify it correctly as
  a platform-level principle inherited from skipper, not rebus's own
  Principle II. The decision it supports (skipper owns no app-level role
  logic) is unchanged. NOTE: the frozen `reference/` docs (015, 017,
  023, …) carry the same skipper-Principle-II framing against the rebus
  constitution path; that is expected drift in frozen snapshots (skipper
  is source of truth) and is intentionally NOT corrected here.

**Unchanged / out of scope:**
- Every Decision Log entry stands as-is.
- The `../features/005-private-chat-1to1.md` link remains dead (the
  feature item doesn't exist yet — `specs/features/` holds only
  `TEMPLATE.md`); creating it is F-005, the owner's gate.
- `rebus`, `bateau`, `kingfish` are referenced by name (no "not yet
  created" markers in this doc); all three repos now exist (created
  2026-06-24/25), consistent with the 006 audit.
- Status stays Complete.
