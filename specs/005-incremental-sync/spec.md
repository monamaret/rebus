# Feature Specification: Incremental Sync

**Feature Branch**: `005-incremental-sync`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "A chat client stays current with a conversation by pulling only what has changed since its last sync — new messages, edits, hard-deletes, and hides — rather than re-fetching the whole conversation each time. Sync is manual-check-only for v1: the client triggers a pull on demand (when the participant checks), with no background polling and no server push. Delivery is asynchronous and pull-only by design; there is no live, real-time, or streaming path. A participant who has been offline catches up fully on reconnect from their last sync point. The client tracks its last-sync position per conversation across pulls."

**Source research** (this spec points back to confirmed decisions; it does not re-decide them):
- [R-006](../research/006-private-chat-application.md) — `updatedAt`-incremental sync extending sight's `SyncNotes(ctx, since)`; manual-check-only for v1; async-only delivery; per-participant `hiddenFor` excluded from a participant's own pull.
- [R-015](../research/reference/015-shell-notification-and-sync-freshness-model.md) — the manual-check / queue-and-flush-on-return freshness model; no real-time interrupt during a session.
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle II (stateless, async-only — no live/push mode), Principle IV (Firestore-backed, incremental-sync data model).

**Depends on**: Feature 2 (messages exist) and Feature 3 (edits advance `last-updated`; hard-deletes physically remove; hides are per-participant) — sync reflects the changes those features produce.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Pull only what changed since the last sync (Priority: P1)

A participant whose client last synced at a known point requests a refresh and receives exactly the messages that changed after that point — new sends, content edits, hard-deletions, and hides — not the entire conversation. The client then advances its last-sync position to the latest change received.

**Why this priority**: Incremental (delta-only) pull is the defining behavior of this feature and the whole reason a "sync" capability exists separately from a plain read; without it, every refresh is a full re-fetch. It is the first independently-testable slice.

**Independent Test**: Can be tested by seeding changes after a known sync point and confirming a pull returns exactly those changes (no more, no less) and advances the cursor — without exercising offline catch-up or hidden-message filtering.

**Acceptance Scenarios**:

1. **Given** a client whose last-sync point is T1, **When** it pulls, **Then** it receives exactly the messages that changed after T1 and nothing that was already synced.
2. **Given** a successful pull, **When** the client records its new sync point, **Then** a subsequent pull with no intervening changes returns nothing new.
3. **Given** a mix of changes (new, edited, deleted, hidden) after T1, **When** the client pulls, **Then** all four change kinds are reflected in the result.

---

### User Story 2 - Manual-check-only, no push (Priority: P2)

The participant explicitly checks for updates (opens the conversation, requests a refresh); the client pulls at that moment. The backend never pushes updates to the client, and the client does not poll in the background while idle. New messages simply wait until the participant next checks.

**Why this priority**: Manual-check-only is a deliberate v1 constraint (async delivery, no real-time mode) that shapes the whole freshness model; it is verified separately so the absence of push/poll is confirmed, not just implied.

**Independent Test**: Can be tested by producing a change, leaving the client idle, and confirming nothing arrives until the participant manually triggers a pull — and that no push channel exists.

**Acceptance Scenarios**:

1. **Given** a new change in the conversation, **When** the client is idle and the participant has not checked, **Then** nothing is delivered to the client.
2. **Given** the participant, **When** they manually request a refresh, **Then** the pending change is pulled at that moment.
3. **Given** the system, **When** its delivery mechanisms are enumerated, **Then** there is no server-push, background-poll, or real-time/streaming path.

---

### User Story 3 - Offline participant catches up on reconnect (Priority: P2)

A participant who was offline (or had the client closed) reconnects and checks. Their client pulls every change since their last sync point — no matter how long the gap — so they end up current with the conversation, including edits and deletions that happened while they were away.

**Why this priority**: Catch-up is the high-value consequence of incremental sync for an async, infrequently-checked channel; it shares P2 with manual-check because it depends on that model.

**Independent Test**: Can be tested by syncing a client, taking it offline, producing many changes over a gap, reconnecting, and confirming one pull brings it fully current.

**Acceptance Scenarios**:

1. **Given** a client offline since T1 and a long gap with many changes, **When** it reconnects and pulls, **Then** it receives all changes in (T1, now] and becomes current.
2. **Given** an edit and a hard-delete that occurred while the participant was offline, **When** they catch up, **Then** the edit is reflected and the deleted message is absent.
3. **Given** a very large gap since the last sync, **When** the client pulls, **Then** it still receives the bounded delta (the changes), not an error and not the entire history anew.

---

### User Story 4 - A participant does not pull their own hidden messages (Priority: P3)

When a participant has hidden a message from their own view (Feature 3), their subsequent syncs do not pull that message for them — the hide is respected on the sync path. The other participant's sync is unaffected and still receives the message.

**Why this priority**: Hide-respect on sync is what makes "hide" a meaningful per-participant state rather than a cosmetic one; it cross-cuts the above stories but is verified directly.

**Independent Test**: Can be tested by hiding a message as one participant, syncing both participants, and confirming the hider does not receive it while the other does.

**Acceptance Scenarios**:

1. **Given** a message a participant has hidden, **When** that participant syncs, **Then** the hidden message is excluded from their pull.
2. **Given** the same hidden message, **When** the other participant syncs, **Then** they still receive it (the hide is per-participant, not global).

---

### Edge Cases

- What happens on the very first sync, when the client has no last-sync point? (Pull the conversation's full history, then begin incremental syncs from the newest change — see Open Questions for whether v1 backfills all history or a bounded window.)
- What happens if the client's stored sync point is lost or rolled back? (A rolled-back cursor re-pulls some already-seen changes; re-pulling is safe and idempotent from the client's view. Lost-cursor handling defaults to a fresh full sync.)
- What happens if a change's recorded time ties exactly with the cursor? (A tie is treated as already-synced; only strictly-later changes are pulled, so nothing is duplicated or dropped.)
- What happens to a hard-deleted message on sync? (It is simply absent — physical removal, not a tombstone the client interprets.)
- What happens if many changes arrive in one pull? (The client applies them in order and advances the cursor once; this spec takes no position on paging — see Open Questions.)
- What about clock skew between backend and client? (The cursor is backend-derived change times, not client wall-clock, so skew does not cause misses.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A client MUST pull only messages whose last-changed time is later than its stored last-sync point (incremental, delta-only).
- **FR-002**: After a successful pull, the client MUST advance its last-sync point to the latest change received.
- **FR-003**: A pull MUST reflect all change kinds produced since the last sync: new messages, content edits, hard-deletions, and hides.
- **FR-004**: An edited message's advanced last-changed time MUST cause it to be re-pulled (never missed because its send time predates the cursor).
- **FR-005**: A hard-deleted message MUST be absent from the pull (physical removal, no tombstone).
- **FR-006**: A participant's own hidden messages MUST be excluded from that participant's pull; the other participant's pull MUST be unaffected.
- **FR-007**: Sync MUST be manual-check-only — triggered by the participant on demand — with NO server push and NO client background polling.
- **FR-008**: Delivery MUST be asynchronous and pull-only; there MUST be no live, real-time, or streaming path.
- **FR-009**: The client MUST persist its last-sync point per conversation across pulls and across restarts.
- **FR-010**: On the first sync (no stored point), the client MUST pull the conversation's history (full back-fill, or a v1 bounded window — see Open Questions).
- **FR-011**: Sync MUST operate per-conversation; each conversation has its own cursor.

### Key Entities *(include if feature involves data)*

- **SyncCursor**: the last-sync point a client holds per conversation — the boundary past which the next pull begins. Persisted client-side; derived from backend change times, not client wall-clock.
- **ChangeSet** (implicit): the delta a pull returns — new, edited, hard-deleted (absent), and hidden-since messages relative to the cursor.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A client whose cursor is T1 and that pulls at T2 receives exactly the messages changed in (T1, T2] — no already-synced messages, no missed changes — in 100% of tested cases.
- **SC-002**: An offline participant catches up to fully current on a single reconnect pull, regardless of gap length, in 100% of tested cases.
- **SC-003**: The system has zero push channels, zero background-poll paths, and zero real-time/streaming paths (verifiable by enumerating delivery mechanisms).
- **SC-004**: A participant's hidden messages are excluded from their own pull and present in the other participant's pull, in 100% of tested cases.
- **SC-005**: After the first sync, each subsequent pull returns a bounded delta (only the changes), never the full conversation history.

## Assumptions

- Messages, edits, hard-deletes, and hides (Features 2 and 3) exist and advance/record the change times this sync reads.
- The backend is the authoritative source of change times; the client's cursor is derived from those, not from client wall-clock, so clock skew does not cause misses.
- The client has durable local storage for its per-conversation sync cursors.
- v1 is async/manual-check-only by constitution (Principle II); introducing push, polling, or real-time is explicitly out of scope and would be a constitution-level change, not a feature.
- Backfill depth on first sync (full history vs. a bounded window) is an Open Question; indefinite retention (R-006) means full history is available, but whether v1 fetches all of it is undecided.
