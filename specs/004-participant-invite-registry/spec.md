# Feature Specification: Participant Invite & Public-Key Registry

**Feature Branch**: `004-participant-invite-registry`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "The owner (admin) admits a participant to the private chat by registering that participant's public SSH key in the owner-controlled registry — no shared domain, no external identity provider, no account system. Registering a key both provisions the participant (so they can authenticate) and establishes a 1:1 conversation between owner and invitee, assigning their roles (owner = admin, invitee = participant). An outside-domain guest is provisioned identically — identity is registry-derived, not domain-derived. The owner can likewise revoke a participant (remove their key), after which that key can no longer authenticate or access the conversation. The registry is the single source of truth for who may participate."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-006](../research/006-private-chat-application.md) — owner-controlled public-key registry; outside-domain provisioning via the registry; multiple independent 1:1 conversations; indefinite retention (messages persist, access is gated).
- [R-013](../research/013-cli-command-mapping-and-privilege-model.md) — invite/revoke are admin-only operations; server-side role enforcement.
- [R-002](../research/reference/002-hybrid-tui-layer.md) — the `admin invite`/`admin revoke-key` (bbb-le) and `register-key` (rook-server-cli) precedent this registry reuses.
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle VI (SSH-Key Identity, IdP-Free), Principle VII (asymmetric roles, the admin owns registry mutation).

**Provides**: the conversation-creation step that Feature 2 (conversation-messaging) assumes already exists, and the registered-key set that Feature 1 (authentication) checks against.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Admin invites a participant (Priority: P1)

The owner registers a participant's public key. This both admits that key to the registry (so the participant can authenticate) and establishes a fresh 1:1 conversation between owner and invitee, with roles assigned (owner = admin, invitee = participant). The invited participant can then authenticate and exchange messages.

**Why this priority**: Invite is the only way a conversation and a participant come to exist — it unblocks Features 1, 2, and 3 for any new participant, so it is the foundational registry operation.

**Independent Test**: Can be tested by registering a previously-unknown key, confirming a conversation now exists between owner and invitee with the correct roles, and that the invitee can authenticate and send/read — without exercising revocation.

**Acceptance Scenarios**:

1. **Given** an unregistered public key, **When** the admin invites it, **Then** the key is admitted to the registry and a 1:1 conversation is established between owner and invitee with roles admin and participant.
2. **Given** a newly invited participant, **When** they authenticate (Feature 1), **Then** authentication succeeds and they can send/read (Feature 2) within that conversation.
3. **Given** a non-admin, **When** they attempt to invite, **Then** the attempt is rejected and no key is registered.

---

### User Story 2 - Admin invites an outside-domain guest (Priority: P2)

The owner invites a participant from outside the owner's domain — someone with no shared account system, no shared identity provider, no prior relationship to any external IdP. Provisioning is identical to an in-domain invite: the guest's registered public key is their entire identity, and they participate purely on the strength of the registry entry.

**Why this priority**: Outside-domain provisioning is what makes this a genuinely private cross-domain 1:1 channel rather than a same-domain tool; it shares the registry mechanism with US1 but is verified separately to confirm identity is registry-derived, not domain-derived.

**Independent Test**: Can be tested by inviting a key whose owner has no account in any external identity provider and confirming the guest authenticates and participates with no IdP dependency.

**Acceptance Scenarios**:

1. **Given** a guest with no account in any external identity provider, **When** the admin registers their public key, **Then** the guest can authenticate and participate purely on the strength of the registry entry.
2. **Given** the invitation and all subsequent participation, **When** the flow is observed, **Then** no call to or dependency on GCP, Google, or any external IdP occurs.
3. **Given** an outside-domain guest and an in-domain invitee, **When** each is provisioned, **Then** the mechanism and resulting capabilities are identical.

---

### User Story 3 - Admin revokes a participant (Priority: P2)

The owner removes a participant's key from the registry. After revocation, that key can no longer authenticate and the revoked participant can no longer access the conversation(s) they were part of. Revocation does not erase conversation history (retention is indefinite per R-006); it removes access.

**Why this priority**: Revocation completes the registry lifecycle and is the owner's safety lever for a compromised or unwelcome key; it shares P2 with outside-domain invite because both are essential-but-secondary to the core invite.

**Independent Test**: Can be tested by inviting a key, then revoking it, and confirming the key can no longer authenticate or read/send — without exercising a fresh invite.

**Acceptance Scenarios**:

1. **Given** a registered participant, **When** the admin revokes their key, **Then** subsequent authentication with that key is rejected and the participant can no longer access the conversation.
2. **Given** a revoked participant, **When** the conversation history is examined, **Then** the history is retained (not erased) — only access is removed.
3. **Given** a non-admin, **When** they attempt to revoke, **Then** the attempt is rejected and the key remains registered.

---

### User Story 4 - Registry is the sole source of truth (Priority: P3)

Only keys the owner has registered may authenticate and participate. An unregistered key is rejected at authentication and can never reach any messaging operation; the registry — not the transport, not the client, not an external IdP — is the single determinant of who may participate.

**Why this priority**: This is the trust guarantee that makes the whole identity model coherent; it cross-cuts the above stories but is verified directly so it is not only implied.

**Independent Test**: Can be tested by attempting authentication and messaging with a key that was never registered and confirming 100% rejection at every layer.

**Acceptance Scenarios**:

1. **Given** a key that has never been registered, **When** it attempts to authenticate, **Then** it is rejected.
2. **Given** the system as a whole, **When** determining whether someone may participate, **Then** the registry is the sole determinant — no other source can admit a participant.
3. **Given** a revoked key, **When** it attempts any operation, **Then** it is treated as unregistered (rejected).

---

### Edge Cases

- What happens when the admin re-invites an already-registered key? (Idempotent or a clear "already invited"; this spec takes no position on whether it re-creates a conversation or reuses an existing one — see Open Questions.)
- What happens when the admin revokes an already-revoked key? (No-op / clear "already revoked"; no error.)
- What happens to a conversation's history when one participant is revoked? (History is retained — retention is indefinite per R-006; the revoked participant simply loses access. Whether the remaining participant sees an indication is a downstream UX question.)
- Can the owner revoke themselves or otherwise leave a conversation? (This spec assumes the owner is persistent; self-revocation is out of scope — see Open Questions.)
- What happens if the owner registers a second device's key for the same person? (That is a separate registry entry; single-device identity is the R-006 default — multi-device is noted in Open Questions.)
- How many participants can a conversation have? (Exactly two — invite establishes a 1:1 channel; a third party cannot be added.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: An admin MUST be able to register a participant's public key (invite), admitting it to the registry.
- **FR-002**: Registering a key MUST establish a 1:1 conversation between owner and invitee with roles assigned (owner = admin, invitee = participant).
- **FR-003**: An outside-domain guest MUST be provisioned identically to an in-domain invitee — identity is registry-derived, not domain-derived.
- **FR-004**: Provisioning and all subsequent participation MUST be IdP-free — no dependency on GCP, Google, or any external identity provider.
- **FR-005**: An admin MUST be able to revoke a participant by removing their key from the registry.
- **FR-006**: After revocation, the revoked key MUST no longer authenticate and the revoked participant MUST no longer access the conversation(s).
- **FR-007**: Revocation MUST NOT erase conversation history (retention is indefinite); it removes access only.
- **FR-008**: Only keys registered in the registry MAY authenticate and participate; unregistered and revoked keys MUST be rejected.
- **FR-009**: Invite and revoke MUST be admin-only operations; non-admins MUST be rejected.
- **FR-010**: Each conversation established by invite MUST contain exactly two participants.
- **FR-011**: Re-inviting an already-registered key and revoking an already-revoked key MUST be handled without error (idempotent or a clear "already applied" outcome).

### Key Entities *(include if feature involves data)*

- **RegisteredPublicKey**: a public key the owner has admitted to the registry, bound to a participant identity and role. The sole determinant of who may participate; the anchor Feature 1 checks against.
- **Conversation** (created here): the 1:1 channel established at invite time between owner and invitee, carrying the two participants' identities, their roles, and a created timestamp. The substrate Feature 2 operates on.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Inviting a key establishes a 1:1 conversation with correct roles AND enables the invitee to authenticate, in 100% of tested cases.
- **SC-002**: An outside-domain guest authenticates and participates with zero calls to or dependencies on any external identity provider, in 100% of tested cases.
- **SC-003**: After revocation, the revoked key fails authentication and is blocked from all conversation operations, in 100% of tested cases, while history is retained.
- **SC-004**: An unregistered key is rejected at authentication and never reaches any messaging operation, in 100% of tested cases.
- **SC-005**: 100% of invite/revocation attempts by a non-admin are rejected with no registry mutation.

## Assumptions

- The owner (admin) identity is established and persistent; the owner performs all registry mutations.
- Public keys are exchanged out-of-band before invite (the owner obtains the invitee's public key through some trusted channel not specified here).
- A single registered key represents one device/identity for one participant; single-device identity is the R-006 default (multi-device and second-key handling are Open Questions, not resolved here).
- Conversation history retention is indefinite (R-006); revocation removes access, not data.
- WebAuthn/passkeys are reserved for a future web client (R-020) and are not part of this registry's v1 surface.
