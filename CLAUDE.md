# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository. Keep it in sync with `AGENTS.md` — that file is the
canonical source of truth; this one must not drift from it (constitution
Governance).

## Project state

`rebus` — the private 1:1 chat app's **backend** (`github.com/monamaret/rebus`),
one application within the platform that `skipper` scaffolds and operates —
does not yet have implementation source. This is the initial agent/workflow
configuration pass. See `AGENTS.md`'s "Build / Test / Lint" section for the
real commands once the Go module exists; that section is the source of truth
and is updated as soon as it does.

`rebus` is a stateless RPC/HTTPS backend (rook/sight pattern), Firestore-backed
(`conversations/{id}/messages/{id}`, `updatedAt`-incremental sync), written in
Go, deployed to GKE by `skipper`, publishing one public Go client package
(`github.com/monamaret/rebus/client`) that `kingfish` and `bateau` import. It
is **not** the platform tool, a TUI shell, or a client.

## Repo layout

- `AGENTS.md` — the canonical project instructions file, auto-loaded by
  opencode (see `opencode.json` -> `instructions`). Treat it as the source of
  truth for project conventions — extend/update it alongside this file rather
  than letting the two drift apart.
- `opencode.json` — opencode config; points `instructions` at `AGENTS.md`.
- `.opencode/agents/` — project-scoped opencode subagent definitions. One
  markdown file per agent; filename = agent name, frontmatter + system-prompt
  body. See `.opencode/agents/README.md` for the exact file format. New agents
  must also be listed under "Subagents" in `AGENTS.md`.
- `.specify/memory/constitution.md` — the non-negotiable project principles
  (single-owner/minimal-dependency bias; stateless client-mediated backend;
  the public client package as a cross-repo contract; Firestore-backed
  incremental-sync data model; transport+at-rest — not E2EE — confidentiality;
  SSH-key IdP-free identity; asymmetric server-side roles; secure-by-default
  GKE deployment via `skipper`; test-first development). Read this before
  proposing architecture or scope changes.
- `specs/research/`, `specs/features/`, `specs/releases/` — the
  research-doc → feature-item → roadmap → architecture-sync → Spec Kit
  pipeline described in `AGENTS.md`'s "Feature development process". The docs
  under `specs/research/` are the **vendored platform corpus copied from
  `skipper`** (settled design context — chiefly `006`/`013`/`014`/`019`/`021`/
  `023`/`026` for `rebus`); `rebus`'s own new research/features extend it.
- `.claude/settings.json` — enabled Claude Code plugins for this repo.
  `.claude/skills/speckit-*/` are Spec Kit's generated command surface, not
  hand-authored plugin config. `.claude/skills/tech-research/` and
  `.claude/skills/release-tracking/` are hand-authored project skills — see
  `AGENTS.md`'s "Skills" section. `.claude/settings.local.json` is
  machine-specific and must never be committed (see the policy note below).
- `CHANGELOG.md` (repo root) — one entry per release, newest first, following
  the scaffold in `specs/releases/CHANGELOG-TEMPLATE.md`. Every line item
  links back to the `specs/features/` item that justified it.

### Local vs. project-level agent configuration

Only project-level, shared agent/workflow configuration is tracked here:
`.claude/settings.json`, `.claude/skills/speckit-*/`, `.specify/`,
`opencode.json`, `.opencode/`. Personal or machine-specific settings —
`.claude/settings.local.json`, editor-specific agent config (`.cursor/`,
`.windsurf/`, `.clinerules/`, etc.), or anything else that varies per
developer/machine rather than per project — must never be committed;
`.gitignore` excludes the common cases preventatively. If you find any of
these tracked in git, remove them; if you need a personal override, keep it
untracked locally instead.

## Working in this repo

- Before writing implementation code, check whether `AGENTS.md` has been
  filled in with real conventions/build commands — it is the file opencode
  loads as project instructions and is meant to be the living guide.
- New capabilities or non-trivial technical decisions start with a research
  doc under `specs/research/`, based on `specs/research/research-template.md`
  — see the "Feature development process" in `AGENTS.md` and the "Decision
  Recording & Research Workflow" in the constitution.
- This repo holds real conversation-service configuration for GCP/GKE and
  Firestore. Never commit secrets, kubeconfigs, or GCP service-account keys —
  see the constitution's Security & Operational Posture section. `rebus`
  authenticates via Workload Identity / Workload Identity Federation, not
  static keys; the SSH-key challenge auth never transmits a private key.
- `rebus` is deployed and operated **by `skipper`** (`skipper app add
  --github`), not by itself. This repo owns the Go source, the container
  build, and the WIF-authenticated CI that pushes to Artifact Registry — the
  persisted Kustomize base and Config Connector IAM/namespace config live in
  `skipper`'s `deployments/rebus/`, not here.
- The public `rebus/client` package is a versioned cross-repo contract
  (`kingfish` and `bateau` compile against it). Never make a breaking change
  to its surface without a SemVer-major release coordinated with those
  importers; never place it under `internal/`.
- Deliberate, recorded scope boundaries — **not** to drift across without a
  constitution-level amendment: transport+at-rest confidentiality (not E2EE),
  indefinite retention, server-side (not client-side) role enforcement,
  1:1-only (no group chat), async-only (no real-time/live mode).

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->
