---
name: tech-research
description: Research a technical implementation question, library choice, or architecture decision for skipper, following the standardized specs/research/ pattern — library/best-fit assessment, existing-implementation survey, and a dedicated Web Citations section. Use when starting research on a new technical decision before it becomes a feature item.
argument-hint: <topic, question, or candidate feature to research>
---

## Purpose

Produce a `specs/research/NNN-<slug>.md` document for `$ARGUMENTS` that
follows skipper's established research pattern (see
`specs/research/research-template.md` and the existing docs `001`–`005`
for worked examples) — not an ad hoc write-up. This skill exists so every
technical research doc in this repo is structurally consistent: same
sections, same rigor on library/pattern assessment, same separation
between internal cross-references and external web citations.

Read `.specify/memory/constitution.md` (especially Principle I —
single-owner, minimal-dependency bias) and
`specs/research/roadmap.md` §0 before starting. If the topic in
`$ARGUMENTS` is already partially covered by an existing research doc,
extend that doc instead of creating a duplicate — check
`specs/research/*.md` first.

## Process

1. **Determine the doc number and slug.** Scan `specs/research/` for the
   highest existing `NNN-` prefix; the new doc is `NNN+1-<slug>.md`, where
   `<slug>` is a short, kebab-case name for the topic.
2. **Copy `specs/research/research-template.md`** as the starting point —
   never start from a blank file or improvise different section headings.
3. **Fill in every section**, in this order, before moving to the next:
   - **Problem Statement** — goal, in/out of scope, success criteria, and
     non-goals. State the actual question being answered, not a
     restatement of the feature idea.
   - **Background & Context** — what's already decided elsewhere that
     bears on this (search other `specs/research/*.md` docs and the
     constitution for relevant prior decisions before writing this).
   - **Libraries & Stack** — for every credible candidate (language,
     framework, library, managed service), fill the comparison table and
     evaluate it against skipper's *actual* stack and constraints (Go/
     TypeScript, GCP, Kubernetes, single-owner bias), not generic best
     practice. **State an explicit "Selected" recommendation** — a
     research doc that surveys options without recommending one is
     incomplete. List every credible rejected candidate with its specific
     reason. Every maturity/license/maintenance claim must trace to a
     citation in the Web Citations section (step 6) — don't assert from
     memory.
   - **Architecture Patterns** — survey applicable design/architecture
     patterns, and explicitly list **known existing implementations** of
     each pattern (open source projects, vendor reference architectures,
     write-ups) rather than describing patterns purely in the abstract. If
     this repo's own reference implementations (biblio, bbb-le,
     rook-reference, sight) are relevant, name the specific mechanism
     being borrowed, not just the repo. Cite external implementations in
     Web Citations.
   - **Existing Codebase Evaluation** — assess this repo and any named
     reference repos already in scope for reuse opportunities, following
     the precedent in docs `002`, `003`, and `005`.
   - **Security & Compliance**, **Performance, Reliability & Operations**
     — fill in whatever is concretely knowable now; don't leave these as
     "Not yet applicable" if there's a real, statable concern (e.g. auth,
     secrets, GCP IAM scoping, rollback mechanics).
   - **Risks & Mitigations** — at minimum one row per non-trivial tradeoff
     surfaced in Libraries & Stack or Architecture Patterns.
   - **Open Questions** — anything that is genuinely the owner's call to
     make, not something you can reason to a confident recommendation for.
     Don't pad this section with questions you could answer yourself from
     the research already done.
   - **Decision Log** — leave empty if nothing is confirmed yet. Never
     invent a decision the owner hasn't actually made.
   - **Internal References** — links to other docs *within this repo*
     only (other research docs, feature items, constitution, architecture.md,
     roadmap.md).
   - **Web Citations** — **every external web source consulted**
     (documentation, benchmarks, articles, vendor docs, GitHub repos not
     already covered under Internal References) goes here, in its own
     table, separate from Internal References. Include the URL and the
     date it was checked. This is mandatory, not optional — a research
     doc that drew on web sources but doesn't list them here is
     incomplete and should not be marked Complete.
4. **Resolve what you can yourself; ask about the rest.** Per this
   project's established working style, give a clear recommendation with
   tradeoffs for technical questions you can reason about from the
   research itself. Only use a clarifying-question tool for decisions
   that are genuinely the owner's to make (product direction, risk
   tolerance, priorities) — don't ask the owner to do the comparative
   analysis work this skill exists to do.
5. **Cross-link.** Add an entry to `specs/research/roadmap.md` §0 if this
   doc introduces a new tracked open-question area; update an existing §0
   entry instead if it narrows one already there (follow the pattern in
   how docs `001`–`005` are cross-referenced from `roadmap.md`).
6. **Set Status accurately.** `In Progress` if open questions remain,
   `Complete` only once every open question is either resolved (with a
   Decision Log entry) or explicitly deferred with a stated reason.

## Constraints

- Never fabricate a citation. If you can't find a credible source for a
  claim, say so in the doc rather than inventing one.
- Never invent a Decision Log entry the owner hasn't actually confirmed —
  recommendations go in "Selected"/"Candidate patterns" prose; the
  Decision Log is reserved for confirmed calls.
- Don't restructure `research-template.md`'s section order or headings
  for an individual doc — if the template itself needs to change to
  support a new kind of research, propose that change to
  `specs/research/research-template.md` directly (and flag it to the
  owner) rather than diverging in just one doc.
