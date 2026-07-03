# `specs/research/` — `rebus`'s research corpus

This directory holds the research corpus for **`rebus`**
(`github.com/monamaret/rebus`), the private 1:1 chat app's stateless
backend. It is split into two physical layers:

- **Top level (this directory) — `rebus`-owned.** The three rebus-core
  design docs (`006`, `013`, `014`), whose subject *is* `rebus`/`bateau`,
  plus the navigation docs (`architecture.md`, `roadmap.md`,
  `research-template.md`, this `README.md`). These are maintained here.
- **[`reference/`](reference/) — vendored platform reference, frozen.** The
  other 23 numbered docs, copied from `skipper`
  (`github.com/monamaret/skipper`). **Source of truth is `skipper`**; these
  snapshots are not updated here. See
  [`reference/README.md`](reference/README.md) for the index and the
  source-of-truth rule.

The numbering (`R-NNN`) is `skipper`'s platform-wide scheme, preserved
verbatim in both layers so an `R-006` citation means the same thing across
the platform's repos. Per the constitution
([`.specify/memory/constitution.md`](../../.specify/memory/constitution.md)):

> Platform-level decisions `rebus` inherits (rook/sight stateless pattern,
> Firestore, SSH-key identity, transport+at-rest confidentiality, asymmetric
> admin/participant roles, per-app namespace/IAM isolation) are recorded in
> the vendored `specs/research/` corpus copied from `skipper` and are cited
> per principle below rather than re-decided here.

## Edit rules

- **`reference/` — read-only.** Do not edit the 23 inherited docs. They are
  frozen snapshots; `skipper` is the source of truth. Drift between these
  copies and `skipper` is expected; `skipper` wins. (Their internal markdown
  links are not maintained — see [`reference/README.md`](reference/README.md).)
- **The three rebus-core docs (`006`/`013`/`014`) — `rebus`-owned, edited in
  place.** Written during `skipper`'s research process but owned and
  maintained here. They are corrected to stay accurate — e.g. applying the
  stale-noun corrections `R-022` prescribed (see PR #1 for `006`). Their
  substantive, dated Decision Log entries are preserved as history; only
  stale nouns/framing get corrected, never the decisions themselves.
- **`rebus`'s own *new* research** extends the corpus with **new** numbered
  docs (copy `research-template.md`), never by rewriting the 23 inherited
  reference docs.

## Top-level docs (rebus-owned)

### Rebus-core — the subject IS `rebus` or `bateau`

| Doc | Topic |
| --- | --- |
| [`006-private-chat-application.md`](006-private-chat-application.md) | **The `rebus` design doc** — confidentiality model, Firestore schema, sync, roles, conversation scope |
| [`013-cli-command-mapping-and-privilege-model.md`](013-cli-command-mapping-and-privilege-model.md) | `rebus`'s asymmetric admin/participant role model (also contains `skipper`'s command table as context) |
| [`014-guest-chat-client-scope-and-auth-ux.md`](014-guest-chat-client-scope-and-auth-ux.md) | `bateau` — the guest's standalone client (commands, distribution, SSH-agent auth) |

### Navigation docs

- [`architecture.md`](architecture.md) — `rebus`'s own architecture: the chat
  backend, Firestore schema, public client package, auth, and GKE deployment.
  Cites the `skipper` platform architecture as inherited context, not as the
  subject.
- [`roadmap.md`](roadmap.md) — `rebus`'s own implementation roadmap;
  `F-005` (rebus's operation/method set) is the next concrete step, with
  `skipper`'s first release as dependency context.
- [`research-template.md`](research-template.md) — the template for new
  research docs (copy it; new `rebus` research extends the corpus with new
  numbering).

## Inherited reference (23 docs, frozen)

All 23 vendored platform docs live in [`reference/`](reference/) — grouped
there by rebus-relevance (inherited context / tangential / platform-only).
**For the full index and the source-of-truth rule, see
[`reference/README.md`](reference/README.md).**
