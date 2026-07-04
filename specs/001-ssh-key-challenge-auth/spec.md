# Feature Specification: SSH-Key Challenge Authentication

**Feature Branch**: `001-ssh-key-challenge-auth`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "Authenticate a private 1:1 chat participant using their SSH private key, with no external identity provider (IdP-free). The backend issues a challenge; the client signs it locally and the private key never leaves the client. The client resolves its key from the SSH agent first, falling back to an encrypted private-key file with a passphrase prompt. Identity is independent of GCP/Google or any external IdP; participants (including an outside-domain guest) are provisioned solely through the owner-controlled public-key registry. This is the foundation for all other chat capabilities."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-002](../research/reference/002-hybrid-tui-layer.md) — SSH-key, IdP-free identity baseline; the signed-challenge pattern.
- [R-014](../research/014-guest-chat-client-scope-and-auth-ux.md) — agent-first / encrypted-key-file-fallback resolution order.
- [R-020](../research/reference/020-webauthn-implementation-mechanics.md) — WebAuthn reserved for a future web client only (out of scope for TUI v1).
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle VI (SSH-Key Identity, IdP-Free).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Authenticate via SSH agent (Priority: P1)

A participant opens the chat client with an SSH agent running and their key loaded. The client requests access; the backend issues a one-time challenge; the client asks the agent to sign it and returns the signature. The backend verifies the signature against the participant's registered public key and establishes the session. The participant never enters a passphrase and the private key never leaves the agent.

**Why this priority**: The agent path is the everyday, friction-free case for a participant who has already loaded their key (e.g. via `ssh-add`). It is the path every returning participant hits on a normal machine, so it must work transparently and fast.

**Independent Test**: Can be fully tested in isolation by presenting a registered key through an agent, issuing a challenge, and confirming a valid signature is accepted and the session is established — without exercising any messaging capability.

**Acceptance Scenarios**:

1. **Given** a participant whose public key is registered, **When** they connect with an agent holding the corresponding private key, **Then** authentication completes and a session is issued, with no passphrase prompt.
2. **Given** a registered participant, **When** their agent signs the backend's challenge, **Then** the private key material is present only on the client/agent and never appears in any backend-bound request.
3. **Given** an agent that holds multiple keys, **When** the participant authenticates, **Then** the key used is the one registered to that participant (or the one the participant selects).

---

### User Story 2 - Authenticate via encrypted key file with passphrase (Priority: P2)

No agent is available (or the needed key is not loaded in it). The client locates the participant's encrypted private-key file, prompts for the passphrase with no echo, decrypts it locally, signs the challenge, and discards the decrypted key material. Authentication then proceeds as in User Story 1.

**Why this priority**: This is the fallback that makes the client usable on a machine without an agent (a fresh machine, a CI context, or simply a user who prefers not to run an agent). It must work but is secondary to the transparent agent path.

**Independent Test**: Can be tested by pointing the client at an encrypted key file, entering the passphrase, and confirming the resulting signature is accepted — independently of the agent path.

**Acceptance Scenarios**:

1. **Given** no usable SSH agent, **When** the participant provides a passphrase, **Then** the encrypted key file is decrypted locally, the challenge is signed, and authentication succeeds.
2. **Given** an encrypted key file, **When** the passphrase is entered, **Then** the prompt does not echo the passphrase, and the decrypted key material is not written to disk or retained beyond signing.
3. **Given** a wrong passphrase, **When** the participant submits it, **Then** authentication fails with a clear error and no key material is exposed.

---

### User Story 3 - Provision and authenticate an outside-domain guest (Priority: P3)

The owner adds an outside-domain participant by registering that participant's public key in the owner-controlled registry (no shared domain, no external IdP, no shared account system). The guest then authenticates via User Story 1 or 2, and their identity is established solely from the registered key — independent of any GCP/Google or third-party identity.

**Why this priority**: Provisioning the guest is what makes this a *private 1:1* conversation rather than a same-domain tool, but it depends on the core auth flows (US1/US2) already working, so it is sequenced after them.

**Independent Test**: Can be tested by registering a previously-unknown key, then authenticating with it, and confirming the identity is accepted purely on the basis of the registry entry.

**Acceptance Scenarios**:

1. **Given** an owner-controlled registry, **When** the owner registers a guest's public key, **Then** that guest can authenticate and a participant who is not registered cannot.
2. **Given** a guest with no account in any external identity provider, **When** they authenticate with their registered key, **Then** their identity is established without any call to or dependency on an external IdP.

---

### Edge Cases

- What happens when an SSH agent is present but does not hold a key matching the participant's registered public key? (Fall through to the key-file path, with no silent failure.)
- What happens when a participant presents a key that is not in the registry? (Reject; the error must not leak whether the key is "unknown" vs. "wrong signature" to avoid registry enumeration.)
- What happens when the same challenge is submitted twice, or an old challenge is replayed? (Each challenge is single-use and expires; a replay is rejected.)
- What happens when the challenge expires before the client signs and returns it? (Reject with a clear "challenge expired, retry" outcome; the client re-attempts transparently.)
- What happens when the key file is unencrypted (no passphrase)? (Sign directly; do not prompt.)
- What happens on repeated wrong-passphrase attempts? (Fail clearly each time; this spec takes no position on lockout/throttling — see Open Questions.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The backend MUST issue a fresh, single-use, time-bounded challenge when a participant requests authentication.
- **FR-002**: The client MUST sign the challenge locally; the private key MUST never be transmitted to or stored by the backend.
- **FR-003**: The client MUST attempt to sign via the SSH agent first when one is available.
- **FR-004**: The client MUST fall back to reading an encrypted private-key file and prompting for a passphrase (no echo) when the agent path cannot produce a signature.
- **FR-005**: Authentication MUST be IdP-free — it MUST NOT depend on GCP, Google, or any external identity provider.
- **FR-006**: A participant's identity MUST be established solely from a public key registered in the owner-controlled registry; an unregistered key MUST be rejected.
- **FR-007**: Challenges MUST be single-use and MUST expire, so that a captured or replayed challenge cannot be used to authenticate.
- **FR-008**: Failed authentication MUST produce a clear, non-enumerable error (the response MUST NOT distinguish "unknown key" from "bad signature" in a way that enables registry enumeration).
- **FR-009**: The participant MUST be able to select which identity/key to use when more than one is available (per R-014's `--identity` convention).
- **FR-010**: Authentication MUST establish a session the client uses for all subsequent operations; this spec covers the authentication surface only (the session's lifetime/shape is refined in downstream operation specs).

### Key Entities *(include if feature involves data)*

- **RegisteredPublicKey**: a public key the owner has admitted to the registry, keyed to a participant identity and a role (admin/participant). Source of truth for "who may authenticate."
- **Challenge**: a single-use, time-bounded nonce the backend issues and later verifies a signature against. Discarded after one verification attempt.
- **ParticipantIdentity**: the identity established from a verified signature over a valid challenge; carries the participant's role for the session.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A returning participant with an agent-loaded key authenticates in a single user-perceived step (one round-trip: challenge → signed response), with no passphrase entry.
- **SC-002**: The private key is verifiably absent from every backend-bound message — confirmable by inspecting that all client→backend auth traffic contains only public material (public key, signature, challenge reference) and never the private key.
- **SC-003**: Authentication succeeds end-to-end with no external identity provider configured or reachable (IdP-free by construction).
- **SC-004**: A registered participant is accepted and an unregistered key is rejected, in 100% of tested cases — identity is registry-derived, not inferred.
- **SC-005**: A replayed or expired challenge is rejected in 100% of tested cases.

## Assumptions

- Each participant has (or generates) an SSH keypair; the owner has registered the corresponding public key before the participant first authenticates.
- v1 is TUI-only; WebAuthn/passkey support is reserved for a future web client (R-020) and is explicitly out of scope here.
- Identity is single-device: one key per participant per device (R-006); registering a second device's key is a separate registry action, not part of this auth flow.
- A secure transport (HTTPS) is already in place; this spec covers the application-level signed-challenge authentication layered on top, not transport encryption itself.
- The backend has a source of time available for challenge expiry.
