# specs/assets

Scaffold placeholder for design/reference assets.

The UI-kit assets bundled in the upstream `skipper` spec repo
(`catalyst-ui-kit`, `commit`, `protocol`, `transmit` — Tailwind Plus kits)
were intentionally **not** copied here. `rebus` is the private 1:1 chat
**backend** (stateless RPC/HTTPS, rook/sight pattern, Firestore-backed);
its v1 clients are the `kingfish` TUI embedded view and the `bateau` guest
TUI, with a web layer explicitly deferred. None of those web UI kits are
relevant to the backend or its Go client package.

Add rebus-specific assets (wire-protocol diagrams, Firestore schema
sketches, TUI mockups) here as the specification work progresses.
