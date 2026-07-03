---
name: release-tracking
description: Before implementing a feature/task, verify it traces to an existing scoped item (specs/features/ entry, roadmap.md entry, or referenced issue) — never let work proceed unscoped. When closing a release, generate the release directory and a standardized CHANGELOG.md entry from the changelog scaffold. Use when starting implementation work or when preparing/closing a release.
argument-hint: <feature/task being implemented, or "close release X.XX">
---

## Purpose

This skill has two jobs, both about keeping implementation work
traceable: (1) gate implementation work on it actually being scoped, and
(2) produce skipper's standardized release documentation and changelog
when a release closes. It does not replace the Spec Kit workflow
(`/speckit.specify` etc.) or the feature-development pipeline in
`AGENTS.md` — it sits alongside them as the relevance/traceability check
and the release-closing step.

## Part 1 — Scope-alignment gate (before/during implementation)

Before writing or modifying implementation code for `$ARGUMENTS`, confirm
it traces to one of:

- a `specs/features/*.md` item (preferred — the established pipeline:
  research doc → feature item → roadmap entry → architecture sync →
  Spec Kit, per `AGENTS.md`'s "Feature development process"),
- an entry in `specs/research/roadmap.md` (if the feature item doesn't
  exist yet but the roadmap entry is concrete and dependency-ordered), or
- a referenced GitHub issue/PR that itself links back to one of the
  above.

**If no such trace exists:** stop before writing implementation code.
Tell the user what's missing and offer to either (a) create the missing
scoping artifact first (a feature item, or a roadmap entry, following the
existing templates), or (b) get explicit confirmation that this is
deliberately out-of-process work (e.g. a trivial fix) before proceeding
anyway. Don't silently proceed on unscoped work and don't silently invent
a feature item just to satisfy this check — the point is to catch
drift, not paper over it.

**While implementing**, keep track of which feature/task ID each actual
change maps to — this is what Part 2 turns into release notes and a
changelog. If work expands beyond what the linked feature item describes,
flag the scope creep rather than quietly broadening the implementation;
either the feature item needs updating first, or the extra work belongs
in a separate, separately-scoped item.

## Part 2 — Release documentation and changelog (closing a release)

When the user asks to prepare or close a release (e.g. "close release
0.1" or "let's cut the first release"):

1. **Verify readiness** per constitution's Release Process: every feature
   item bundled in the release must actually be shipped (its Scope
   checklist complete, acceptance criteria met) and have passing unit +
   integration tests. Don't generate release documentation for
   unfinished work — flag what's incomplete instead.
2. **Create the release directory** `specs/releases/X.XX-name/` containing
   exactly what the constitution requires:
   - **Final feature list** — every shipped feature item in this release,
     each entry linking to its `specs/features/` file.
   - **Release notes** — plain, owner-facing summary of what changed and
     why (no marketing framing, per Principle I).
   - **Test evidence** — pointer to or summary of the unit/integration
     tests covering this release's capabilities.
   - **Usage guide** — how to scaffold, configure, deploy, upgrade, and
     maintain a project with this release, per Principle II's full
     lifecycle.
3. **Add a changelog entry.** Copy the block from
   `specs/releases/CHANGELOG-TEMPLATE.md` into `CHANGELOG.md`, directly
   under the `# Changelog` heading (newest entry first), and fill it in:
   - One line item per shipped feature, under the right category (Added/
     Changed/Fixed/Security/Deferred) — every line item links to its
     `specs/features/` item.
   - Include a **Deferred** entry for anything that was originally
     in-scope for this release but got pushed out (e.g. a feature whose
     web client was deferred) — link to wherever that deferral is
     recorded (the research doc's Decision Log, or the roadmap).
   - Update the `## [Unreleased]` section at the top of `CHANGELOG.md` to
     reflect whatever's next, or remove it if nothing is currently in
     flight.
4. **Do not fabricate shipped status.** If a feature item isn't actually
   done, it doesn't go in the changelog or the final feature list — flag
   it as remaining work instead.

## Constraints

- Never let implementation work proceed without a traceable scope — this
  is the whole point of Part 1. A quick fix is fine to flag and proceed
  on with confirmation; silent scope drift is not.
- Never invent release notes, a final feature list, or changelog entries
  for work that hasn't actually shipped and been tested.
- Keep changelog entries plain and owner-facing (Principle I) — this is
  not a public-facing marketing changelog.
- Use `specs/releases/CHANGELOG-TEMPLATE.md` as the scaffold for every
  entry — don't improvise a different changelog format per release.
