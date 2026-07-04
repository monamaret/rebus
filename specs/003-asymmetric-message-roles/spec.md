# Feature Specification: Asymmetric Message Roles

**Feature Branch**: `003-asymmetric-message-roles`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "The owner (admin) and the guest (participant) have asymmetric powers over messages in a 1:1 conversation, enforced by the backend. The admin can hard-delete a message (physically remove it so neither participant sees it) and edit a sent message's content (the edit is visible to both participants and distinguishable from a new message). The participant can only hide a message from their own view (a soft hide that never removes the message and never affects the other participant's view) and cannot edit or delete anything. The role is checked at the backend on every operation, never in the clients. Admin operations surface only in the owner's client, never in the guest's."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-013](../research/013-cli-command-mapping-and-privilege-model.md) — the asymmetric admin/participant privilege model; server-side enforcement; admin operations never reach the guest client.
- [R-006](../research/006-private-chat-application.md) — `hiddenFor` per-participant soft-hide; `updatedAt` distinct from `sentAt` so an edit is distinguishable from a new message in sync; hard-delete (physical removal) vs. hide (soft).
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle VII (Asymmetric Roles, Enforced Server-Side).

**Depends on**: Feature 2 (conversation-messaging) — messages must exist before they can be mutated; the per-participant role carried on each conversation (set in Feature 2) is the input this feature enforces.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Admin hard-deletes a message (Priority: P1)

The owner removes a message from the conversation entirely. After deletion, neither the owner nor the guest can see or read that message; it is physically gone rather than hidden, so the deletion propagates to both participants' views.

**Why this priority**: Hard-delete is the admin's primary moderation lever and the clearest expression of the role asymmetry — it is the operation only the admin can perform and the one that most needs to be correct, so it is tested first.

**Independent Test**: Can be tested by seeding a message, hard-deleting it as admin, and confirming both participants' subsequent reads no longer return it — without exercising edit or hide.

**Acceptance Scenarios**:

1. **Given** a conversation with a message, **When** the admin hard-deletes it, **Then** neither participant's subsequent read returns that message.
2. **Given** a hard-deleted message, **When** either participant reads the conversation, **Then** the message is absent (not blanked, not marked deleted — physically removed).
3. **Given** a participant attempting the admin's hard-delete, **When** they invoke it, **Then** it is rejected and the message remains.

---

### User Story 2 - Admin edits a sent message (Priority: P1)

The owner changes the content of a message that has already been sent. Both participants subsequently see the new content, and the edited message is distinguishable from a brand-new message (the original send time is preserved while the "last-updated" time advances, so sync treats it as an edit, not a fresh message).

**Why this priority**: Edit is the second admin-only operation and shares the same role-enforcement surface as hard-delete; it is the operation that makes the `updatedAt`-vs-`sentAt` distinction load-bearing, so it is verified alongside delete.

**Independent Test**: Can be tested by sending a message, editing it as admin, and confirming both participants read the new content while the message remains attributable to its original send time — without exercising delete or hide.

**Acceptance Scenarios**:

1. **Given** a sent message, **When** the admin edits its content, **Then** both participants' subsequent reads return the new content.
2. **Given** an edited message, **When** it is read, **Then** its original send time is preserved and its "last-updated" time is later than the send time (so the edit is distinguishable from a new message).
3. **Given** a participant attempting the admin's edit, **When** they invoke it, **Then** it is rejected and the content is unchanged.

---

### User Story 3 - Participant hides a message from their own view (Priority: P2)

The guest (or the owner, on their own view) soft-hides a message. The hide removes the message from that one participant's view only; the message itself remains and the other participant's view is completely unaffected. The participant cannot edit or delete the message.

**Why this priority**: Hide is the participant's only message-level power and demonstrates the "soft, self-only" end of the asymmetry; it depends on the admin operations being defined first to make the contrast testable, so it follows them.

**Independent Test**: Can be tested by seeding a message, hiding it as one participant, and confirming that participant no longer sees it while the other still does — without exercising delete or edit.

**Acceptance Scenarios**:

1. **Given** a message in a conversation, **When** one participant hides it, **Then** that participant's subsequent reads do not return it, but the other participant's reads still do.
2. **Given** a hidden message, **When** the hiding is applied, **Then** the message is not removed — it is marked hidden for that one participant only.
3. **Given** a participant, **When** they attempt to edit or hard-delete any message, **Then** the attempt is rejected (their only message-level power is hide).

---

### User Story 4 - Backend rejects privilege violations (Priority: P3)

A participant (or any non-admin) attempts an admin-only operation (edit or hard-delete). The backend rejects it regardless of what the client sends — role is enforced server-side, so a client that fails to gate the operation cannot perform it. Admin operations are never even surfaced in the guest client.

**Why this priority**: This is the security guarantee that makes the whole asymmetry trustworthy; it cross-cuts the above stories but is called out separately so it is verified directly rather than only as a side-effect.

**Independent Test**: Can be tested by issuing edit/hard-delete requests with participant credentials and confirming 100% rejection, and by confirming the admin operation surface is absent from the guest client.

**Acceptance Scenarios**:

1. **Given** a request to edit or hard-delete, **When** the caller's role is not admin, **Then** the backend rejects it without mutating any message.
2. **Given** the guest client, **When** its operation surface is enumerated, **Then** edit and hard-delete are not present (admin operations surface only in the owner's client).
3. **Given** any message operation, **When** it executes, **Then** the role check happens at the backend, independent of any client-side gating.

---

### Edge Cases

- What happens when the admin hard-deletes an already-deleted message, or edits an already-edited one? (Delete is idempotent-ish (no-op / clear "already gone"); a re-edit advances "last-updated" again and preserves the original send time.)
- What happens when a participant hides an already-hidden message? (Remains hidden for that participant; idempotent from their view; the other participant is never affected.)
- What happens to the "last-updated" timestamp on an edit? (Advances past the send time; this is what lets the sync path treat it as an edit rather than missing it — see Feature 5.)
- What happens if a hard-deleted message is later referenced (e.g. by an older client's sync cursor)? (It is simply absent; hard-delete is physical removal, not a tombstone the client must interpret.)
- What happens to hide state if the same participant uses two devices? (Hide is per-identity/per-view as recorded; single-device identity is assumed per R-006 — see Open Questions for the multi-device nuance.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: An admin MUST be able to hard-delete a message, physically removing it so neither participant's subsequent reads return it.
- **FR-002**: An admin MUST be able to edit a sent message's content such that both participants subsequently see the new content.
- **FR-003**: An edited message MUST preserve its original send time while advancing its "last-updated" time, so the edit is distinguishable from a new message.
- **FR-004**: A participant MUST be able to hide a message from their own view only; the hide MUST NOT remove the message or affect the other participant's view.
- **FR-005**: A participant MUST NOT be able to edit or hard-delete any message; their only message-level power is hide.
- **FR-006**: Every message operation MUST have the caller's role checked at the backend; role MUST NOT be enforced only in the clients.
- **FR-007**: A non-admin issuing an admin-only operation (edit or hard-delete) MUST be rejected, with no message mutation.
- **FR-008**: Admin operations (edit, hard-delete) MUST surface only in the owner's client; they MUST NOT appear in the guest client's operation surface.
- **FR-009**: Re-applying an already-applied state (hide an already-hidden message; delete an already-deleted one) MUST be handled without error (idempotent or a clear "already applied" outcome).
- **FR-010**: Hard-delete MUST physically remove the message (no tombstone); hide MUST mark it hidden for one participant only (soft).

### Key Entities *(include if feature involves data)*

- **Role**: per-participant, per-conversation — `admin` (owner) or `participant` (guest). The value this feature enforces; set in Feature 2.
- **Message** (extended): gains a per-participant hidden-marker (the soft-hide) and the edit semantics that advance its "last-updated" time while preserving its send time.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A hard-deleted message is absent from both participants' reads in 100% of tested cases; a hidden message is absent only from the hiding participant's reads, in 100% of tested cases.
- **SC-002**: An edit is visible to both participants and its "last-updated" time strictly follows its send time, in 100% of tested cases.
- **SC-003**: 100% of edit and hard-delete attempts by a non-admin are rejected with no message mutated.
- **SC-004**: The admin operation surface (edit, hard-delete) is absent from the guest client's enumerated operations in 100% of inspections.
- **SC-005**: Re-applying hide/delete to an already-hidden/already-deleted message produces no error and no change to the other participant's view.

## Assumptions

- Messages and conversations already exist (Feature 2) and each conversation carries a per-participant role; this feature enforces that role on mutation.
- The authenticated identity (Feature 1) reliably determines the caller's role for a given conversation.
- "Hide" is a per-participant, per-view soft state; single-device identity is assumed (R-006). The multi-device hide-sync nuance is noted in Open Questions, not resolved here.
- The backend is the sole mutation authority; clients request mutations and the backend applies (or rejects) them — consistent with the stateless, backend-mediated model (constitution Principle II).
- v1 has no group/multi-party conversations, so "admin"/"participant" always describe a two-person conversation's two roles (Principle VII).
