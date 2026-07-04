# Feature Specification: Public Client Package

**Feature Branch**: `006-public-client-package`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "The backend publishes a single client library that the two separate client apps (the owner's embedded view and the guest's standalone client) import to reach every backend capability — authentication, messaging, role-gated operations, invite/registry, and sync. The library is importable across separate codebases (not locked to one), exposes a uniform, cancellable operation shape with typed, handlable errors, and is versioned: additive changes do not break existing importers, and any breaking change is a coordinated major-version release. Both clients depend on this one library rather than each re-implementing the protocol."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-026](../research/reference/026-public-client-package-contract.md) — one public (not internal) package per backend; the `Client`/constructor/wire-types/error-sentinels surface; SemVer-major on breaking change.
- [R-006](../research/006-private-chat-application.md) — the shared client-library decision (one protocol, two clients, no duplication); the bounded wire-type package precedent.
- [R-023](../research/reference/023-platform-repo-inventory.md) — `kingfish` and `bateau` are separate repos that import this package across repo boundaries.
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle III (the public client package is a cross-repo contract).

**Consumes**: the operations defined by Features 1–5 (auth, messaging, role-gated ops, invite/registry, sync) — this feature packages them as one importable, versioned surface. The concrete method roster (exact operation names and inputs/outputs) is finalized downstream (F-005), not enumerated here.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Two clients share one library to reach every capability (Priority: P1)

The owner's client (full admin) and the guest's client (send/read/hide only) are separate apps in separate repos, yet both reach the backend by importing the same single client library. Neither re-implements the protocol; the library is the one shared path for authentication, messaging, role-gated operations, invite/registry, and sync.

**Why this priority**: A single shared library consumed by two independent clients is the defining property of this feature and the reason it exists as more than an internal helper; without it, the two clients diverge or duplicate the protocol. It is the first independently-testable slice.

**Independent Test**: Can be tested by importing the library from two separate codebases and confirming each can invoke the operations it is entitled to (admin surface for the owner's client; participant surface for the guest's) through the one library — without exercising versioning.

**Acceptance Scenarios**:

1. **Given** the owner's client and the guest's client in separate repos, **When** each imports the library, **Then** both reach the backend's capabilities through that one library and neither duplicates protocol logic.
2. **Given** the library, **When** its surface is enumerated, **Then** it exposes the operations from Features 1–5 (auth, messaging, role-gated ops, invite/registry, sync).
3. **Given** the two clients, **When** the owner's client uses the admin operations and the guest's uses only participant operations, **Then** both do so through the same library (the role difference is enforced server-side, not by shipping two libraries).

---

### User Story 2 - Additive changes do not break importers (Priority: P2)

The backend grows — a new operation is added, or an operation gains a new optional input. Both existing importers keep building and working against the updated library without code changes. Only a change that breaks the existing surface forces a coordinated major-version release.

**Why this priority**: Additive-safety is what lets the backend evolve without forcing both clients to update in lockstep; it shares P2 with cross-repo importability because both are what make a shared library preferable to two private ones.

**Independent Test**: Can be tested by adding a new operation and a new optional input to the library, then confirming both importers build and run unchanged.

**Acceptance Scenarios**:

1. **Given** an importer built against the current library, **When** a new operation is added, **Then** the importer still builds and works without modification.
2. **Given** an importer, **When** an existing operation gains a new optional input, **Then** the importer's existing calls remain valid and unchanged.
3. **Given** a change that alters or removes existing surface, **When** it is proposed, **Then** it is NOT shipped as additive — it requires a major-version release coordinated with both importers.

---

### User Story 3 - Importable across separate codebases (Priority: P2)

The library is importable by codebases other than the backend's own — it is not restricted to the backend repo or locked behind a mechanism that forbids cross-repo use. Both `kingfish` and `bateau` import it from their own repos as a normal dependency.

**Why this priority**: Cross-repo importability is the structural requirement that makes "two clients, one library" possible at all; it is verified separately so the absence of a cross-repo barrier is confirmed.

**Independent Test**: Can be tested by importing the library from a fresh, separate codebase and confirming the import resolves and the operations are callable.

**Acceptance Scenarios**:

1. **Given** a client repo separate from the backend, **When** it declares the library as a dependency, **Then** the import resolves and the operations are usable.
2. **Given** the library, **When** its packaging is examined, **Then** it is exposed as a public, cross-repo-importable surface (not an internal/restricted one).

---

### User Story 4 - Outcomes are handlable via typed errors (Priority: P3)

When an operation fails (not-found, unauthorized, conflict, etc.), the library returns distinguishable, typed errors so each importer can branch on the outcome programmatically rather than parsing strings. Both clients handle the same error kinds consistently.

**Why this priority**: Typed errors are what make the library ergonomic to build correct client behavior on; it cross-cuts the above but is verified directly so error-handling correctness is not left implicit.

**Independent Test**: Can be tested by triggering each error kind through the library and confirming each is distinguishable by type/predicate, not by message text.

**Acceptance Scenarios**:

1. **Given** an operation that fails (e.g. unauthorized, not-found, conflict), **When** the importer handles the result, **Then** it can distinguish the failure kind programmatically.
2. **Given** the same failure kind, **When** either client handles it, **Then** both branch on the same typed error consistently.

---

### Edge Cases

- What happens when the backend ships a breaking change? (It MUST be a major-version release, coordinated with both importers before/with the release — not a silent change either importer discovers at build time.)
- What happens if an importer stays on an old major version while the backend moves on? (The old major version remains importable; cross-major is the importer's explicit choice, not a forced break.)
- What happens if a new operation is admin-only? (It is part of the library surface, but the backend enforces the role; the guest client simply does not call it — the library does not ship two variants.)
- What happens to backend internals? (They MUST NOT be exposed by the library; only the public operation surface and its types/errors are exported.)
- Does the library expose transport/protocol internals? (No — importers program against operations and types, not the wire format; the protocol is an implementation detail owned by the library.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The backend MUST publish exactly one client library.
- **FR-002**: Both client apps (owner's and guest's) MUST reach the backend's capabilities by importing that one library, without duplicating protocol logic.
- **FR-003**: The library MUST be importable by codebases separate from the backend repo (not restricted/internal to it).
- **FR-004**: The library MUST expose the operations from Features 1–5 (authentication, messaging, role-gated operations, invite/registry, and sync) as a uniform, cancellable operation surface.
- **FR-005**: The library MUST return typed, distinguishable errors for failure kinds (e.g. not-found, unauthorized, conflict), handlable without parsing messages.
- **FR-006**: The library MUST expose only the public operation surface and its types/errors; it MUST NOT expose backend internals or raw transport/protocol details.
- **FR-007**: Additive changes (new operations, new optional inputs) MUST NOT break existing importers.
- **FR-008**: Any change that alters or removes existing surface MUST be a coordinated major-version release.
- **FR-009**: The library MUST be versioned (semantic versioning), and importers pin against a known version.
- **FR-010**: A single library MUST serve both the admin (owner) and participant (guest) operation subsets; role differences are enforced server-side, not by shipping separate libraries.

### Key Entities *(include if feature involves data)*

- **ClientLibrary**: the one published, versioned, cross-repo-importable package exposing the backend's operations. Versioned with semantic versioning; the contract both clients pin against.
- **Operation**: a uniform, cancellable call on the library surface (the packaging of each capability from Features 1–5). Each takes typed inputs and returns typed outputs or a typed error.
- **TypedError**: a distinguishable, programmable failure kind the library returns (not-found, unauthorized, conflict, etc.).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Two separate client codebases import the one library and each reaches its entitled operations (admin for the owner, participant for the guest) through it, in 100% of tested cases.
- **SC-002**: An additive change (new operation, new optional input) ships and both existing importers build and run unchanged, in 100% of tested cases.
- **SC-003**: 100% of surface-breaking changes are gated on a coordinated major-version release; zero break to either importer arrives as a non-major change.
- **SC-004**: The library is importable from a fresh, separate codebase as a normal dependency, in 100% of tested cases (no cross-repo import barrier).
- **SC-005**: Each failure kind is distinguishable by type/predicate rather than message text, in 100% of tested cases.

## Assumptions

- The backend operations packaged here (Features 1–5) exist and are the surface this library exposes.
- Both importers (`kingfish`, `bateau`) are separate repos/modules that declare this library as a dependency (R-023).
- The library is language-native to the backend and its clients (the concrete language, package path, and call syntax are R-026 / plan.md decisions, not specified here).
- Versioning follows semantic versioning; importers are expected to pin and to coordinate ahead of major-version uptake.
- The concrete method roster (exact operation names and input/output shapes) is finalized downstream (F-005) and is explicitly out of scope for this specification, which fixes the *contract properties*, not the method list.
