# Deep-research verification plan — `specs/research/` rebus-owned docs

**Created:** 2026-07-03
**Owner:** mona
**Method:** Verify & strengthen in place (not re-litigate). For each note,
re-run the `tech-research` skill's rigor against the current state of the
world: refresh Web Citations and their `Accessed` dates, confirm decisions
still hold, fill citation gaps, fix broken/stale links, and add a dated
audit trail. **Decisions are preserved** — the pass only escalates (flags
to the owner) if verification proves a recorded decision is now technically
wrong; it never silently rewrites one.

## Scope (5 files)

The three rebus-owned research docs plus the two navigation docs. The 23
vendored docs under `specs/research/reference/` are **out of scope** — they
are frozen snapshots (`skipper` is source of truth) and are read-only here
(see `specs/research/reference/README.md`). They are *read* during this
pass (to verify cross-references) but never *edited*.

| # | File | Role | Why this dependency order |
| --- | --- | --- | --- |
| 1 | `specs/research/006-private-chat-application.md` | **THE design doc** | Foundational. 013 and 014 both resolve follow-ups 006 flagged; architecture.md and roadmap.md synthesize it. Verify first; everything downstream cites it. |
| 2 | `specs/research/013-cli-command-mapping-and-privilege-model.md` | Privilege convention | Generalizes 006's admin asymmetry; 014 depends on it (bateau has no admin role). Verify after 006, before 014. |
| 3 | `specs/research/014-guest-chat-client-scope-and-auth-ux.md` | `bateau` scope/auth | Resolves 006's follow-ups and respects 013. Has the weakest Web Citations section — most to gain. Verify after 006 + 013. |
| 4 | `specs/research/architecture.md` | Synthesis (diagrams/flows) | Re-draws 006/013/014/026. Verify after the three primary docs are strengthened so it inherits corrections, not before. |
| 5 | `specs/research/roadmap.md` | Status / what's next | Depends on everything above. Verify last so it reflects the strengthened corpus. |

## Cross-cutting rules (apply to every note)

1. **`reference/` is read-only.** Never edit the 23 frozen docs. All edits
   land in the 5 in-scope files only.
2. **Decisions are preserved.** This pass adds citations, refreshes
   `Accessed` dates, fills gaps, fixes links, and records an audit trail.
   It does NOT re-decide. If verification shows a recorded decision is now
   technically wrong, **flag it to the owner** in the audit note + a new
   Open Question — do not silently rewrite the Decision Log.
3. **Every new or refreshed citation gets `Accessed = 2026-07-03`** (today).
4. **Never fabricate.** Per the `tech-research` skill: if a source can't be
   found or a private repo isn't accessible, say so honestly in the doc
   rather than inventing or hand-waving a citation.
5. **One note at a time, gated.** Each note is worked alone; its checkpoint
   (below) must pass before the next note starts. The checkpoint is an
   owner review of the diff for that one file.
6. **Audit trail per doc.** Each strengthened doc gets a dated
   **"Verification audit — 2026-07-03"** entry (in its Appendix, or a new
   subsection at the bottom if no Appendix) recording what was checked,
   what changed, and what was left as-is. This preserves the doc's history
   per the rebus-core edit rules in `specs/research/README.md`.
7. **Status line stays `Complete`** unless a decision is found wrong; an
   audit pass alone doesn't revert it.

---

## Note 1 — `006-private-chat-application.md` (the design doc)

**Dependency rationale:** Foundational. Everything downstream cites it.

### Citations to re-verify (Web Citations table, all currently `2026-06-24`)

- `https://firebase.google.com/docs/firestore/manage-data/structure-data` —
  backs the quoted "chat application… organize users and messages as
  collections nested within chat room documents" line and the subcollection
  pattern. Firebase docs get restructured; confirm the URL resolves and the
  quote is still present, else find the canonical new location and update
  both URL and quote.
- `https://firebase.google.com/docs/firestore/query-data/queries` — backs
  three load-bearing claims: (a) `>` timestamp queries for incremental
  sync, (b) the single-inequality-field limit, (c) equality filters like
  `hiddenFor` not counting against that limit. **This is the technical
  crux of the sync design** — verify the wording carefully; Firestore
  query docs have shifted over time.
- The three **private-repo** citations (sight `.specify/memory/constitution.md`,
  sight `internal/domain/note_service.go`, rook-reference
  `specs/architecture/component-overview.md`) — all marked "fetched earlier
  in this research thread" and the repos were "not yet created" at write
  time. Check (now, 2026-07-03) whether these repos/paths actually exist
  and whether the quoted strings (`SyncNotes(ctx, customerID, since time.Time)`,
  the `sight-cli` `internal/rpc`-only import rule, `stash/<space-id>/`) are
  really present. If a repo is private/inaccessible/nonexistent, mark the
  citation honestly unverifiable per the never-fabricate rule — do not
  leave it reading as if it were checked.

### Decisions to re-validate against current reality

- Composite index requirement for `updatedAt > X AND hiddenFor == Y`. The
  doc asserts this is supported and backs it only via the query-data doc.
  Firestore has a dedicated **composite indexes** doc — add it as a
  stronger citation.
- "Indefinite retention" (Decision Log). Firestore imposes no expiry that
  contradicts this, but the doc asserts it bare; consider adding the
  Firestore limits doc as a citation so the claim traces somewhere.
- `hiddenFor` field design (per-participant soft-hide). Light check that
  no newer Firestore idiom (Map, array-contains) is now clearly preferred
  for this shape — if not, leave as-is.

### Gaps to fill

- Add the Firestore **composite indexes** doc URL.
- Add the Firestore **subcollection/limits** doc URL where the 1 MiB cap
  and subcollection behavior live (currently referenced only via 009).

### Cross-doc consistency

- 006 links into `reference/` (002, 005, 007, 009) and to `014`.
  Per `reference/README.md`, rebus-core → reference links are supposed to
  be kept current. Open each and confirm the cited claim is actually
  present at the target (e.g. 002's bbb-le / rook-server-cli precedent;
  022's kingfish-repo amendment referenced inline).
- Verify the inline "amended 2026-06-24, see 022 → Decision Log" anchors
  resolve.

### Checkpoint (exit criteria)

1. Every Web Citation URL resolves, or is replaced with the current
   canonical URL with `Accessed = 2026-07-03`.
2. The two load-bearing Firestore claims each trace to a live, quoted
   source.
3. Private-repo citations either verified or honestly marked unverifiable.
4. New citations (composite indexes, limits) added with today's date.
5. "Verification audit — 2026-07-03" entry added to the Appendix.
6. Status stays `Complete` unless a decision is found wrong.

---

## Note 2 — `013-cli-command-mapping-and-privilege-model.md` (privilege convention)

**Dependency rationale:** Generalizes 006's admin asymmetry; 014 depends
on it. Verify after 006.

This doc explicitly states "No new external research was needed" — its Web
Citations section is empty by design. Verification is therefore almost
entirely **internal consistency**, not web research.

### Command table (Part 1) — "the authoritative cross-reference"

Every row's `Decided in` citation must be opened and the claimed command/
tool invocation confirmed present:

- Rows citing `003`, `005`, `007`, `010`, `012` — open each reference doc,
  confirm it actually decides what the row claims.
- The inline **"Updated 2026-07-03"** note added `skipper site new` (017)
  and `skipper tui new` (022). Verify both rows against reference/017 and
  reference/022 specifically.
- Confirm the two "decided-necessary but not yet designed" commands
  (`skipper app remove`, `skipper app list`) are still absent and still
  listed in Open Questions (i.e. the table is still honestly incomplete).

### Privilege convention (Part 2)

- Cites 002 (bbb-le/rook-server-cli precedent), 011 (per-adapter auth
  enforcement), 006, and constitution Principle I & II. Open each and
  confirm the cited claim is present and says what's asserted.
- "bbb-le's `bbb admin invite`/`admin revoke-key`" and "rook-server-cli"
  are named as precedents with **no citation**. If those repos are now
  accessible, add internal-reference links; if not, note as internal
  precedent (honest).

### Open Questions — confirm still open

- `skipper app remove`/`app list` gap — still open? (Can only confirm
  local state; `skipper` is the source of truth and out of scope.)
- Invite-action UI within the TUI-embedded view — still open?

### Gaps to fill

- None forced. Don't over-research an internal-convention doc; the pass is
  a consistency check.

### Checkpoint

1. Every row in the command table verified against its cited reference doc;
   any drift recorded in the audit note.
2. All cross-references into `reference/` resolve.
3. Open Questions confirmed still-open (or updated if resolved).
4. "Verification audit — 2026-07-03" entry added.

---

## Note 3 — `014-guest-chat-client-scope-and-auth-ux.md` (`bateau`)

**Dependency rationale:** Resolves 006's follow-ups; respects 013
(bateau has no admin role). Verify after both.

This doc has the **weakest Web Citations section** of the five — one
`pkg.go.dev` link plus a "Web search — no single URL" entry. The
`tech-research` skill specifically prohibits that pattern. Most to gain.

### Citations to re-verify / replace

- `https://pkg.go.dev/golang.org/x/crypto/ssh/agent` — confirm resolves,
  API unchanged.
- **Replace** the "Web search — Go SSH agent fallback to private key file
  pattern (no single URL)" entry with real citations:
  - `golang.org/x/crypto/ssh` — verify `ParsePrivateKeyWithPassphrase` /
    `ParseRawPrivateKeyWithPassphrase` still exist with those names on
    `pkg.go.dev` (the x/crypto SSH API has evolved). Cite the package doc
    URL with a short quote.
  - `golang.org/x/term` `ReadPassword` — cite `pkg.go.dev`.
  - An authoritative source for the "agent-first, key-file-fallback"
    pattern (OpenSSH docs or a canonical Go SSH example) — currently
    backed only by an uncited "web search."

### Decisions to re-validate

- Auth resolution order (agent → key file → passphrase prompt → clear
  error). Confirm `golang.org/x/crypto/ssh` still supports both PEM- and
  OpenSSH-native encrypted keys as claimed.
- `goreleaser` distribution — cited as "biblio's own distribution decision
  (already cited platform-wide)" but **not cited here**. If biblio's
  decision is in the vendored corpus, add the internal link; else this is
  an uncited platform-precedent claim to flag.
- Cobra, Bubble Tea, `XDG_CONFIG_HOME` — named as platform precedent
  (bbb-le CLI shape, rook-cli config). Internal-precedent claims are fine
  without web citations, but link the reference repos where possible.
- `--identity`/`-i` flag "mirroring bbb-le" — verify bbb-le actually has
  this if the repo is accessible.

### Gaps to fill (the bulk of the work here)

- The doc asserts `ParsePrivateKeyWithPassphrase` /
  `ParseRawPrivateKeyWithPassphrase` handle "both PEM-encrypted and
  OpenSSH-native encrypted private keys" — a specific technical claim
  that must trace to the package doc per the skill. Add the precise URL,
  `Accessed` date, and a quote.
- Cobra and Bubble Tea currently have **no citation at all** — add project
  homepages to Web Citations (don't assert from memory).
- `goreleaser` — add project URL.

### Cross-doc consistency

- Links to 006, 013, and `reference/002` — resolve and confirm cited
  claims present.

### Checkpoint

1. The "no single URL" citation replaced with ≥1 real URL.
2. Every named library (`x/crypto/ssh`, `x/crypto/ssh/agent`, `x/term`,
   Cobra, Bubble Tea, goreleaser) has a Web Citations entry with a live
   URL + `Accessed = 2026-07-03`.
3. Auth-API claims (function names, encrypted-key support) verified
   against current `pkg.go.dev`.
4. "Verification audit — 2026-07-03" entry added.

---

## Note 4 — `architecture.md` (synthesis)

**Dependency rationale:** Re-draws 006/013/014/026. Verify **after** the
three primary docs are strengthened so it inherits corrections.

### Verification scope

- Every claim cites a research doc (R-002, R-006, R-011, R-013, R-019,
  R-020, R-021, R-023, R-026). Open each and confirm it still says what
  architecture.md claims — **especially** R-019/R-021 namespace + SA
  specifics, and R-026 client-package contract.
- **Diagram ↔ Data Model section consistency.** The diagram shows `role[]`
  on the conversation doc; the Data Model section says "per-participant
  `role` field." Confirm both match 006/013's actual decision (an explicit
  per-participant role, not "creator = admin").
- The composite-index claim is **repeated here** — keep it consistent with
  however Note 1's verification settled it in 006.
- Principle citations (II, IV, V, VI, VII, VIII) — verify each cited
  principle actually says what's claimed (constitution numbering confirmed
  I–VIII+).

### Checkpoint

1. All R-NNN citations resolve to the claimed content.
2. Diagram and Data Model section mutually consistent and consistent with
   006/013.
3. Principle citations accurate.
4. No new decisions introduced — if verification surfaces a real change,
   escalate to the primary doc (006/013/014), not here.
5. "Verification audit — 2026-07-03" entry added.

---

## Note 5 — `roadmap.md` (status / what's next)

**Dependency rationale:** Depends on everything above. Verify last so it
reflects the strengthened corpus.

### Verification scope

- **§0 Outstanding design questions** — F-005 method roster, unread-state
  UI, local export format. Confirm all still open. (As of 2026-07-03,
  `specs/features/` holds only `TEMPLATE.md`, so no feature item exists
  yet — confirm this is still true at execution time.)
- **§1 First release items** — confirm the planned filenames/dependencies
  still match reality: no Go module exists yet, F-005 is still the gate.
- "skipper's first release defines six platform features" and "rebus is
  feature 005 of skipper's first release" — frozen-reference claims;
  verify against reference docs and confirm not contradicted.
- Dependency-context section accuracy.

### Checkpoint

1. Every claim about rebus's current state (no module, no feature items,
   F-005 is next) confirmed true at execution time.
2. Cross-links resolve.
3. If any outstanding question was actually resolved *by* this
   verification pass (e.g. a citation now answers part of F-005's shape),
   note it — but **do not pre-resolve F-005**; that's the owner's gate.
4. "Verification audit — 2026-07-03" entry added.

---

## When the pass is done

- All 5 files carry a `2026-07-03` audit entry.
- Every Web Citation is live with a current `Accessed` date; no "no single
  URL" placeholders remain.
- No decision has been silently rewritten; any flagged-as-wrong decision
  is recorded as a new Open Question for the owner.
- Optional closeout: a one-line summary added to root `CHANGELOG.md` only
  if the owner considers this verification pass a releasable change (it
  likely isn't — it's doc hygiene, not a shipped capability; see
  `release-tracking` skill scope).
