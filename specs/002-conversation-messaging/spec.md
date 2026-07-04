# Feature Specification: Conversation Messaging

**Feature Branch**: `002-conversation-messaging`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "A participant in a private 1:1 conversation sends a message, and both participants read the messages exchanged. Each conversation is a dedicated 1:1 channel between exactly two participants (keyed by their registered public keys); multiple independent 1:1 conversations coexist. A message records who sent it, when, and its content. The backend mediates all access — clients never touch the message store directly. This is the core messaging capability on which the role-gated operations (edit/delete/hide), invite, and incremental-sync capabilities build."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-006](../research/006-private-chat-application.md) — Firestore `conversations/{id}/messages/{id}` data model; backend-mediated access; multiple independent 1:1 conversations; single-device identity.
- [R-013](../research/013-cli-command-mapping-and-privilege-model.md) — the per-participant `role` carried on each conversation (used here only to identify the two participants; role-gated *operations* are a separate feature).
- [R-026](../research/reference/026-public-client-package-contract.md) — the `func(ctx, In) (Out, error)` operation shape the send/read methods follow.
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle IV (Firestore-backed data model), Principle II (stateless, client-mediated).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Send a message to a 1:1 conversation (Priority: P1)

A participant who is one of the two members of a conversation composes a text message and sends it. The message is recorded against that conversation with its sender, send time, and content, and becomes available to be read by both participants.

**Why this priority**: Sending is the irreducible purpose of a chat backend. Without it nothing else (reading, syncing, editing) has a subject to act on, so it is the first independently-testable slice.

**Independent Test**: Can be fully tested by sending a message into a two-participant conversation and confirming it is persisted with the correct sender, timestamp, and content — without exercising read, sync, or any role-gated operation.

**Acceptance Scenarios**:

1. **Given** a conversation with two registered participants, **When** one participant sends a message, **Then** the message is stored against that conversation bearing the sender's identity, a send timestamp, and the exact content.
2. **Given** a participant who is NOT a member of a conversation, **When** they attempt to send into it, **Then** the send is rejected.
3. **Given** a new message, **When** it is stored, **Then** it carries a "last-updated" timestamp equal to its send time (so a later admin edit is distinguishable from an original send in the sync path — see Feature 5).

---

### User Story 2 - Read a conversation's messages (Priority: P1)

Either participant of a conversation reads the messages that have been sent into it, in the order they were sent, each showing who sent it, when, and its content.

**Why this priority**: Reading is the counterpart of sending and the other irreducible half of a chat backend; a conversation you can write to but not read is useless. It shares P1 with sending.

**Independent Test**: Can be tested by seeding a conversation with known messages and confirming a read returns exactly those messages in send order with correct attribution — without exercising sending or any mutation.

**Acceptance Scenarios**:

1. **Given** a conversation containing several messages, **When** a participant reads it, **Then** they receive the messages in send order, each with its sender, send time, and content.
2. **Given** a participant who is NOT a member of the conversation, **When** they attempt to read it, **Then** the read is rejected.
3. **Given** two participants in the same conversation, **When** each reads it, **Then** both see the same set of messages.

---

### User Story 3 - Maintain multiple independent 1:1 conversations (Priority: P2)

The same participant may be a party to several distinct 1:1 conversations, each with exactly one other participant. Messages sent in one conversation never appear in or affect another; each conversation is an isolated two-person channel.

**Why this priority**: Multi-conversation is what makes the app usable for more than one relationship at a time, but it is layered on top of the single send/read core, so it follows rather than precedes it.

**Independent Test**: Can be tested by creating two conversations sharing one common participant, sending a message in each, and confirming each read returns only its own conversation's messages.

**Acceptance Scenarios**:

1. **Given** a participant in two separate conversations, **When** they list/read their conversations, **Then** each is distinct and messages do not cross between them.
2. **Given** any conversation, **When** it is examined, **Then** it contains exactly two participants — never one, never three.
3. **Given** the same pair of participants, **When** they converse, **Then** they do so within their conversation's own scope, isolated from every other conversation either of them has.

---

### Edge Cases

- What happens when a participant sends an empty or whitespace-only message? (This spec rejects empty content; max length is a downstream concern — see Open Questions.)
- What happens when a participant sends to a conversation that does not exist or has been removed? (Reject.)
- What happens when a read is requested for a conversation that has no messages yet? (Return an empty result, not an error.)
- What happens when a third party (not one of the two participants) attempts to send or read? (Reject — only the two members can act; this feature does not introduce group/multi-party conversations.)
- What happens to the "last-updated" timestamp when a message is sent? (Set equal to the send time; it diverges from the send time only on a later edit — Feature 3.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A participant MUST be able to send a text message into a conversation of which they are one of exactly two members.
- **FR-002**: A stored message MUST record its sender, a send timestamp, and its content.
- **FR-003**: A stored message MUST carry a "last-updated" timestamp that, on send, equals the send timestamp.
- **FR-004**: Either participant of a conversation MUST be able to read its messages in send order.
- **FR-005**: The backend MUST mediate all send/read access; clients MUST NOT access the message store directly.
- **FR-006**: Send and read MUST be restricted to the two members of a conversation; any non-member MUST be rejected.
- **FR-007**: Each conversation MUST contain exactly two participants; this feature MUST NOT allow adding a third.
- **FR-008**: Multiple independent conversations MUST coexist, and a message sent in one MUST NOT appear in any other conversation.
- **FR-009**: An empty/whitespace-only message MUST be rejected; a read of an empty conversation MUST return an empty result rather than an error.
- **FR-010**: Read MUST return messages in send order with sender, send time, and content for each.

### Key Entities *(include if feature involves data)*

- **Conversation**: a dedicated 1:1 channel between exactly two participants, each identified by their registered public key; carries a created timestamp and a per-participant role (the role is set here but only *enforced* by the separate role-gated-operations feature).
- **Message**: a single unit of content in a conversation, carrying sender, content, a send timestamp, and a "last-updated" timestamp (equal on send; diverges only on a later edit).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A message sent by one participant is readable (with correct sender, timestamp, and content) by both participants of the conversation.
- **SC-002**: 100% of send/read attempts by a non-member are rejected.
- **SC-003**: A participant with N independent conversations can read each in isolation, and a message sent in one appears in zero other conversations, in 100% of tested cases.
- **SC-004**: Every conversation examined contains exactly two participants, never more or fewer.
- **SC-005**: A read of a conversation returns its messages in send order, deterministically, across repeated reads.

## Assumptions

- Participants are already authenticated (Feature 1) and their public keys are registered before a conversation is created.
- Conversation creation itself (establishing the two-participant channel) is handled by the invite/registry feature (Feature 4); this feature assumes a conversation already exists with exactly two members.
- Content is plain text; rich media, formatting, and attachments are out of scope for v1 (R-006).
- A secure transport (HTTPS) and at-rest encryption are already in place; this spec covers the messaging operations, not the confidentiality envelope (R-006 / constitution Principle V).
- Role-gated message mutations (edit, hard-delete, hide) are deliberately out of scope here — they are Feature 3; this feature only carries the per-participant role forward without enforcing role-based mutation.
