# Research: App-catalog document contract — storage, schema, and versioning

**Status:** Complete — recommendation recorded (GCS single JSON object);
owner ratification of the GCS-vs-Firestore call pending
**Date:** 2026-07-03
**Owner:** mona

## Problem Statement

`R-007` ([007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md))
settled the **per-entry** catalog interface schema — five flat types
(`ssh-passthrough` {host, port, target}, `embedded-view` {viewID},
`cli-exec` {binaryPath}, `web` {url}, `none`, the last never written). But the
**container** around those entries was never specified: where the catalog
document lives, its top-level shape, and how it is versioned. `R-002`
([002-hybrid-tui-layer.md](002-hybrid-tui-layer.md)) left storage as a literal
"Cloud Storage **or** Firestore" without picking one. This is a cross-repo
interface — `skipper` writes it (`app add`/`app set-interface`), `kingfish`
reads it (catalog client, cache-with-fallback) — so both sides must agree on
one concrete contract before either is built. `IMPLEMENTATION-PLAN.md §5 Q2`
flags this exact gap as blocking `app set-interface` and all `kingfish` reads.

- **Goal:** Pick the catalog's storage primitive, define its top-level
  document schema (wrapping `R-007`'s per-entry schema), and define its
  versioning, narrow enough to drive both the `skipper` write path and the
  `kingfish` read/cache path.
- **In scope:** GCS-vs-Firestore selection; top-level document shape;
  schema versioning; write-path concurrency; read-path/caching interaction
  with `R-002`'s already-decided rook-style cache-with-fallback.
- **Out of scope:** The per-entry interface schema (settled in `R-007`, reused
  verbatim here); the local cache file format on the `kingfish` side beyond
  "mirrors the fetched document" (an implementation detail of `kingfish`);
  the catalog bucket/project **name** (deployment config — owner sets, tracked
  in `IMPLEMENTATION-PLAN.md §5 Q2`).
- **Success criteria:** A confirmed storage choice, top-level schema, and
  versioning rule recorded in this doc's Decision Log.
- **Non-goals:** A queryable catalog service or per-entry partial-update API —
  the catalog is small and read wholesale; see Architecture Patterns.

## Background & Context

- **Relevant prior work:**
  - `R-002` → Decision Log: catalog is a **durable static artifact** (GCS or
    Firestore), not a running service; every client mirrors `rook-cli`'s
    `cache/spaces.json` cache-with-fallback pattern (render from last-fetched
    local copy, refresh opportunistically, fall back on read failure). That
    decision stands; this doc only resolves *which* artifact and its shape.
  - `R-007` → Decision Log: the five per-entry interface types, flat per-type
    fields, "matching biblio's flat-config precedent." Reused unchanged.
  - `R-009` ([009-markdown-stash-application.md](009-markdown-stash-application.md))
    → already used Firestore's **1 MiB per-document cap** as the reason to
    split stash content (GCS) from metadata (Firestore). The same cap is
    relevant here as a sizing signal (see Libraries & Stack).
- **Domain assumption:** The catalog holds one entry per interfaced app.
  Realistic v1 scale is single-digit-to-tens of entries; even a generous
  future is hundreds — kilobytes of JSON, far under any size limit.
- **Stakeholders:** mona (sole owner; writes the catalog via `skipper`, reads
  it via `kingfish`/`bateau`-adjacent clients).

## Libraries & Stack

| Candidate | Purpose | Maturity | Fit | Notes |
| --- | --- | --- | --- | --- |
| **Cloud Storage (single JSON object)** | Catalog store | GA, GCP-managed (Principle V) | **Best** | One `catalog.json` object; strong read-after-write & list consistency (see Web Citations); mirrors rook's `cache/spaces.json` *file* shape directly; no schema/collection modeling. |
| Cloud Firestore (single doc, or doc-per-entry) | Catalog store | GA, GCP-managed | Weaker for this shape | Query/partial-update strengths are unused here (catalog read wholesale); 1 MiB doc cap and 1,500-byte indexed-field cap are non-issues at this scale but signal a document-DB modeling overhead GCS avoids. |
| A running registry service | Catalog store | — | Rejected in `R-002` | Already rejected as a single point of failure; not reopened. |

- **Selected:** **Cloud Storage, one JSON object (`catalog.json`).** The
  catalog is small, read as a whole, and written rarely by a single admin
  path — exactly a file, not a database. GCS's strong read-after-write /
  read-after-overwrite / list consistency (Web Citations) means an
  `app set-interface` write is immediately visible to any client that fetches
  fresh, with no eventual-consistency window to design around. This matches
  the `cache/spaces.json` *file* shape `R-002`/`R-007` already point at, and
  keeps the write path a plain read-modify-write of one object (Principle I).
- **Rejected (with reason):** **Firestore** — its differentiators
  (per-entry queries, partial-field updates, real-time listeners) are all
  things this catalog does not need; adopting it means modeling a
  collection/document layout and carrying a document-DB dependency for what
  is functionally a static config file. Firestore stays the right tool for
  the app *data stores* that do need it (`pocket` metadata, `rebus` messages),
  not for the catalog. **Doc-per-entry in either store** — rejected because
  clients read the whole catalog anyway (cache-with-fallback caches the whole
  thing), so per-entry granularity buys nothing and complicates atomic reads.
- **Open questions:** the catalog object's bucket/path *name* is deployment
  config, tracked in `IMPLEMENTATION-PLAN.md §5 Q2`, not decided here.

## Architecture Patterns

- **One object, whole-document read/write.** The catalog is a single GCS
  object. `kingfish` fetches the entire object and caches it locally
  (rook-style); `skipper` writes it by read-modify-write of the whole object.
  No partial-update or query API is exposed — deliberately, per Principle I.
- **Top-level document schema:**

  ```json
  {
    "schemaVersion": 1,
    "generatedAt": "2026-07-03T00:00:00Z",
    "entries": [
      { "name": "soft-serve",  "interface": "ssh-passthrough",
        "host": "wishlist.example", "port": 2222, "target": "soft-serve" },
      { "name": "stash",       "interface": "embedded-view", "viewID": "stash" },
      { "name": "dashboard",   "interface": "cli-exec", "binaryPath": "saratoga" },
      { "name": "some-web-app", "interface": "web", "url": "https://..." }
    ]
  }
  ```

  Each `entries[]` element is exactly `R-007`'s per-type shape (only the fields
  relevant to its `interface` value; `none`-interface apps are never written).
  `name` is the unique key. `schemaVersion` is an integer; `generatedAt` is an
  RFC 3339 timestamp of the last write (informational — freshness is the
  client's cache concern, not a field clients gate on).
- **Write-path concurrency: GCS generation preconditions.** Writes are
  read-modify-write, so `skipper` sets `x-goog-if-generation-match` (the GCS
  optimistic-concurrency precondition) to the generation it read, and retries
  the read-modify-write on a precondition failure. For a single-owner tool
  concurrent writes are unlikely, but the precondition is a one-line guard
  against a lost update (e.g. two terminals) and costs nothing.
- **Read-path/caching: do not rely on public object caching.** GCS's one
  consistency caveat is that *publicly cached* objects can serve stale copies
  for up to ~60 min (Web Citations). The catalog is therefore read via the
  authenticated GCS API (not served as a public, CDN-cached asset), so clients
  always see the current object; `kingfish`'s own rook-style local cache is
  the intended (and only) staleness layer, fully under the client's control
  per `R-002`.
- **Versioning rule:** `schemaVersion` is a single integer bumped only on a
  breaking change to the document/entry shape. A client reading a
  `schemaVersion` newer than it understands renders from its last-good cache
  and surfaces a "catalog schema newer than this client — upgrade" notice
  rather than mis-parsing. Additive, optional fields do **not** bump the
  version (forward-compatible by ignoring unknown fields).
- **Tradeoffs:** A whole-object rewrite per catalog change is trivially cheap
  at this scale and buys atomic, consistent reads; it would be the wrong
  choice only at a scale (thousands of frequently-mutating entries) this
  single-owner platform will not reach.
- **Integration points:** `skipper`'s `app add`/`app set-interface` write
  path (`R-003`/`R-013`); `kingfish`'s catalog client read/cache path
  (`R-002`/`R-007`).
- **Data model implications:** none beyond the schema above; the per-entry
  model is `R-007`'s, unchanged.

## Existing Codebase Evaluation

No `skipper` catalog code exists yet — greenfield. `rook-reference`'s
`cache/spaces.json` is the closest existing shape for both the on-disk cache
and the whole-document read model (cited in `R-002`); it is a reference
pattern only, not vendored.

## Security & Compliance

- The catalog object is **not public**. Read and write both go through
  authenticated GCS access, scoped by the per-app/per-tool GCP IAM model in
  `R-021` ([021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md)):
  `skipper`'s write path needs object write on the catalog bucket; a reading
  client (`kingfish`) needs object read only. No new credential type.
- The catalog contains connection metadata (hosts, ports, view IDs), not
  secrets — but keeping it non-public still avoids leaking the platform's app
  topology, consistent with Principle VI (least exposure).

## Performance, Reliability & Operations

- **Load:** negligible — a few-KB object, read on shell launch + manual
  refresh (`R-007`'s launch-only cadence), written only on `app add`/
  `set-interface`.
- **Reliability:** GCS strong consistency means no read-your-write gap after a
  catalog change; the rook-style client cache (`R-002`) covers a GCS outage
  (stale-but-usable). Both failure modes are the graceful-degradation ones
  `R-002` already accepted.
- **Rollout/rollback:** because the object is versioned by GCS generation,
  object versioning on the bucket (optional) gives a trivial rollback of a
  bad catalog write; not required for v1.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| Concurrent `app set-interface` writes lose an update (two terminals) | Low | Low–Medium | GCS `if-generation-match` precondition + read-modify-write retry | mona |
| A future `kingfish` reads a newer `schemaVersion` and mis-parses | Low | Medium | Integer `schemaVersion` + "render from cache, warn, don't mis-parse" rule above | mona |
| Catalog accidentally served as a public, cached asset → stale reads (60-min GCS cache caveat) | Low | Medium | Read via authenticated GCS API only; never expose the object publicly/CDN-cached | mona |

## Open Questions

- [ ] Catalog bucket/object **name** and GCP project — deployment config,
  tracked in `IMPLEMENTATION-PLAN.md §5 Q2`, not a research blocker. — owner: mona
- [ ] Whether to enable GCS object versioning on the catalog bucket for
  point-in-time rollback (nice-to-have, not v1-blocking). — owner: mona

## Decision Log

- **2026-07-03** — **The app catalog is a single JSON object (`catalog.json`)
  in Cloud Storage, not Firestore.** — Rationale: the catalog is small, read
  wholesale, and written rarely by one admin path — functionally a config
  file, for which GCS's strong read-after-write/overwrite/list consistency and
  plain read-modify-write are a better fit than a document DB's query/partial-
  update machinery, which goes unused here (Principle I). Firestore remains
  correct for the app data stores that need it (`pocket`, `rebus`), not the
  catalog. Resolves `R-002`'s deferred "GCS or Firestore" choice. — Decided
  by: mona *(recommendation; owner ratification pending)*.
- **2026-07-03** — **Top-level schema is `{ schemaVersion, generatedAt,
  entries[] }`, each entry exactly `R-007`'s five-type per-entry shape;
  `none`-interface apps are not written.** `schemaVersion` is an integer
  bumped only on breaking changes; unknown/newer versions cause a client to
  render from cache and warn, never mis-parse; additive optional fields don't
  bump it. — Rationale: wraps the already-settled per-entry schema with the
  minimum envelope needed for versioning and freshness, no more. — Decided by:
  mona.
- **2026-07-03** — **Writes use GCS generation preconditions
  (`if-generation-match`) with read-modify-write retry; reads use the
  authenticated GCS API (the object is never public/CDN-cached).** — Rationale:
  cheap optimistic-concurrency guard against lost updates, and avoids GCS's
  only consistency caveat (stale public-cache reads) by keeping the object
  private; `kingfish`'s rook-style local cache remains the sole, client-owned
  staleness layer per `R-002`. — Decided by: mona.

## Internal References

- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — catalog-as-static-
  artifact + rook-style cache-with-fallback decision this doc concretizes;
  source of the deferred "GCS or Firestore" choice
- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) — the five per-entry interface types reused verbatim as `entries[]`
- [003-backend-application-deployment.md](003-backend-application-deployment.md) and [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) — the `app add`/`app set-interface` write path
- [009-markdown-stash-application.md](009-markdown-stash-application.md) — prior use of Firestore's 1 MiB cap as a storage-split signal
- [021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md) — per-tool IAM scoping the catalog's read/write access follows
- [023-platform-repo-inventory.md](023-platform-repo-inventory.md) — `skipper` (writer) and `kingfish` (reader) roles
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle I (minimal dependency), Principle V (GCP-managed alignment), Principle VI (least exposure)

## Web Citations

| Title | URL | Accessed | Relevance |
| --- | --- | --- | --- |
| Cloud Storage — Consistency | https://docs.cloud.google.com/storage/docs/consistency | 2026-07-03 | GCS gives strong read-after-write, read-after-overwrite, and bucket/object list consistency; the only caveat is stale reads of *publicly cached* objects (up to ~60 min) — the basis for choosing GCS and for reading the catalog via the authenticated API rather than as a public cached asset. |
| Firestore — Usage and limits (quotas) | https://firebase.google.com/docs/firestore/quotas | 2026-07-03 | Firestore's 1 MiB max document size and 1,500-byte indexed-field cap — non-issues at catalog scale, but part of why Firestore's document-DB modeling is unnecessary overhead for a small, read-wholesale catalog vs. a single GCS object. |

## Appendix

None yet.
