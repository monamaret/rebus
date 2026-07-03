# Feature: <Title>

**Status:** Draft | Ready for Spec Kit | In Progress | Shipped
**Date:** YYYY-MM-DD
**Owner:** <name>
**Feature ID:** F-NNN  *(feature-item sequence; see the cross-reference
numbering convention in [AGENTS.md](../../AGENTS.md) — `F-NNN` ≠ the research
doc `R-NNN` of the same number)*

> A feature item **points back to already-confirmed research decisions** — it
> does not re-decide anything inline (see the "Feature development process" in
> [AGENTS.md](../../AGENTS.md)). If a genuine open question surfaces here,
> resolve it in a research doc's Decision Log first, then link it, rather than
> settling it in this file.

## Summary

One or two sentences: what capability this delivers and for whom.

- **Goal:**
- **User/system need:**

## Source research

The research doc(s) and specific Decision Log entries this item is built on.
Every design claim below should trace to one of these, not to fresh reasoning.

- [R-NNN <title>](../research/NNN-*.md#decision-log) — what it settled
- [roadmap.md §N](../research/roadmap.md) — this item's dependency-ordered slot
- [architecture.md](../research/architecture.md) — the call-flow(s) it touches

## Scope

- **In scope:**
- **Out of scope:** (with the research/Decision-Log reference that deferred it)
- **Non-goals:**

## Design reference

The confirmed shape, stated as pointers, not re-derivations. Name the repos,
commands, schemas, and patterns this item assembles — each linked to the doc
that decided it.

- **Repos touched:** (per [R-023 platform inventory](../research/023-platform-repo-inventory.md))
- **`skipper` command(s):** (per [R-013 command mapping](../research/013-cli-command-mapping-and-privilege-model.md))
- **Deployment/config artifacts:** (e.g. `deployments/<app>/` Kustomize base)
- **Cross-repo interfaces relied on:** (catalog schema, client-package API, etc.)

## Dependencies & sequencing

- **Depends on:** (feature items / research decisions / environmental
  prerequisites that must land first — e.g. a live cluster, Config Connector)
- **Blocks:**

## Acceptance criteria

Concrete, testable statements. Each should map to at least one test below.

- [ ]
- [ ]

## Test Strategy

**Required — constitution Principle VII (Test-First, NON-NEGOTIABLE):** tests
are written *before* implementation and MUST be confirmed to fail first. This
section is filled in before any code for this feature is written; a feature is
not "Ready for Spec Kit" until it is.

- **Unit test targets:** the units under test and what each asserts (e.g.
  golden-file tests asserting generated Kustomize bases match committed
  fixtures under `testdata/`).
- **Integration test targets:** the end-to-end path(s) exercised, and the real
  vs. stubbed boundaries (e.g. stubbed `kubectl` invocation vs. a live-cluster
  apply gated on environment).
- **Fails-first evidence:** how the failing-test-first requirement is
  demonstrated for this feature (the test that fails before implementation and
  passes after).
- **Test seams / testability notes:** anything about this feature's design that
  is hard to test, and the seam that makes it testable.

## Open questions

Unresolved items that block this feature, each with an owner. Prefer resolving
in a research Decision Log and linking it, rather than deciding here.

- [ ] Question — owner — target resolution

## Internal References

- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — the principles this feature must satisfy
- (research docs, other feature items, architecture.md, roadmap.md)
