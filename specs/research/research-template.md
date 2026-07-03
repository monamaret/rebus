# Research: <Task Title>

**Status:** Draft | In Progress | Complete
**Date:** YYYY-MM-DD
**Owner:** <name>

## Problem Statement

Describe the software development task or question being researched. Capture the goal, the user or system need, the constraints, and the success criteria. Link to any related tickets, RFCs, or prior research.

- **Goal:**
- **In scope:**
- **Out of scope:**
- **Success criteria:**
- **Non-goals:**

## Background & Context

Summarize what is already known. Include prior art, prior decisions, and any constraints imposed by the surrounding system, team, or business.

- **Relevant prior work:**
- **Domain assumptions:**
- **Stakeholders:**

## Libraries & Stack

Evaluate candidate languages, runtimes, frameworks, libraries, and services. Every candidate MUST be evaluated against rebus's actual stack and constraints (Go, GCP/Firestore, GKE, single-owner/minimal-dependency bias per constitution Principle I) — not against generic best practices alone. Every candidate's maturity/maintenance/license claims must trace to a cited source — add it to **Web Citations** below, don't assert it from memory.

| Candidate | Purpose | License | Maturity | Maintenance | Fit | Notes |
| --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |

- **Selected:** State the recommended best fit explicitly, even if not yet confirmed by the owner — a research doc with no recommendation is incomplete.
- **Rejected (with reason):** Every credible candidate that didn't make the cut, with the specific reason it lost.
- **Open questions:**

## Architecture Patterns

Survey the architecture and design patterns that apply, including **known existing implementations of each pattern** — open source projects, vendor reference architectures, or write-ups that already solve this problem or a close analog. Cite every such implementation in **Web Citations** below; don't describe a pattern in the abstract when a concrete existing implementation can be pointed to instead. Note constraints such as latency, throughput, consistency, deployment topology, and operational concerns. Include diagrams where helpful.

- **Candidate patterns:**
- **Existing implementations surveyed:** (project/repo, what it does, what's reusable vs. not — see [005-generic-app-scaffold.md](005-generic-app-scaffold.md) or [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) for the shape this section should take)
- **Selected pattern(s):**
- **Tradeoffs:**
- **Integration points:**
- **Data model implications:**

## Existing Codebase Evaluation

Assess the current codebase (this repo, and any named reference repos already in scope — e.g. biblio, bbb-le, rook-reference, sight) for suitability. Identify modules/patterns to reuse, refactor, or replace. Note code health, test coverage, and coupling.

- **Relevant modules:**
- **Reuse opportunities:**
- **Refactor candidates:**
- **Replacements required:**
- **Code health signals (lint, type, test, complexity):**
- **Conventions to follow:**

## Security & Compliance

Review authentication, authorization, data handling, secrets management, and applicable compliance requirements. For `rebus`, this includes the SSH-key-signed-challenge auth model, the asymmetric server-side role enforcement, Firestore access scoping via the backend service account, Workload Identity (no static keys) for the GKE deployment, Secret Manager for any application secrets, and the transport+at-rest (not E2EE) confidentiality posture.

- **Threat surface:**
- **Data classification:**
- **Compliance regimes:**
- **Mitigations:**

## Performance, Reliability & Operations

Capture expected load, SLOs, scaling characteristics, observability, and rollout strategy. For cluster/deployment-affecting work, capture rollback mechanics explicitly (`kubectl`/`helm`/Terraform rollback path).

- **Expected load:**
- **SLOs / SLIs:**
- **Observability:**
- **Rollout / migration plan:**
- **Rollback plan:**

## Risks & Mitigations

Enumerate risks across technical, product, and organizational axes. Pair each risk with a mitigation and an owner.

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

## Open Questions

List unresolved questions that block progress, with owners and target resolution dates.

- [ ] Question — owner — due date

## Decision Log

Record significant decisions, the date, the decision maker, and the rationale.

- **YYYY-MM-DD** — Decision — Rationale — Decided by

## Internal References

Link to other docs within this repo: related research docs, feature items, the constitution, architecture.md, roadmap.md. Do not put external web URLs here — those belong in Web Citations below.

- [Title](relative-path) — relevance

## Web Citations

**Every external web source consulted during this research** — documentation, benchmarks, articles, discussions, repos not already covered under Internal References, vendor docs, etc. — goes here, in its own section, separate from Internal References. This is mandatory: a research doc that drew on web sources but doesn't list them here is incomplete. Web content changes or disappears, so record when it was checked.

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
|  |  | YYYY-MM-DD |  |

## Appendix

Any additional artifacts: raw notes, benchmark results, sketches, or extended comparisons.
