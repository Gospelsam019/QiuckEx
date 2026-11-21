# Internal Event System

Decoupled event handling using `@nestjs/event-emitter`.

## Registered Events

### Core Payment Events

| Event Name | Description | Payload Schema |
| :--- | :--- | :--- |
| `username_claimed` | Fired when a username is registered | `{ username: string, publicKey: string, timestamp: string }` |
| `payment_received` | Fired when a payment is detected | `{ txHash: string, amount: string, sender: string }` |

### Recurring Payment Events

| Event Name | Description | Payload Schema |
| :--- | :--- | :--- |
| `recurring.link.created` | Fired when a recurring payment link is created | `{ linkId: string, username?: string, destination?: string }` |
| `recurring.link.updated` | Fired when a recurring link is updated | `{ linkId: string, changes: object }` |
| `recurring.link.cancelled` | Fired when a recurring link is cancelled | `{ linkId: string, username?: string, destination?: string }` |
| `recurring.link.paused` | Fired when a recurring link is paused | `{ linkId: string, username?: string, destination?: string }` |
| `recurring.link.resumed` | Fired when a recurring link is resumed | `{ linkId: string, username?: string, destination?: string }` |
| `recurring.link.completed` | Fired when a recurring link completes all periods | `{ linkId: string, totalExecuted: number }` |
| `recurring.payment.due` | Fired 24h before a payment is scheduled | `{ linkId: string, executionId: string, amount: number, asset: string, periodNumber: number }` |
| `recurring.payment.executed` | Fired when a recurring payment succeeds | `{ linkId: string, executionId: string, transactionHash: string, periodNumber: number }` |
| `recurring.payment.failed` | Fired when a recurring payment fails | `{ linkId: string, executionId: string, failureReason: string, retryCount: number, permanent: boolean }` |

### Receipt Reconciliation Events

| Event Name | Description | Payload Schema |
| :--- | :--- | :--- |
| `receipt_status_conflict` | Fired when an observation would regress a receipt's canonical status | `{ receiptId: string, txHash: string, currentStatus: string, observedStatus: string, source: string, observedAt: string }` |

## Receipt Status Transitions

Receipt reconciliation enforces monotonic status transitions at the persistence boundary. A late or duplicated chain event must never overwrite a newer receipt state with an earlier status.

### Canonical status ordering

Statuses are ranked from least to most advanced:

| Rank | Status | Meaning |
| :--- | :--- | :--- |
| 0 | `pending` | Receipt observed but not yet confirmed |
| 1 | `confirmed` | Transaction included in a block |
| 2 | `finalized` | Transaction reached finality |
| 3 | `failed` | Terminal failure state |

### Allowed transitions

| From | Allowed To |
| :--- | :--- |
| `pending` | `confirmed`, `finalized`, `failed` |
| `confirmed` | `finalized`, `failed` |
| `finalized` | *(none — terminal)* |
| `failed` | *(none — terminal)* |

Any transition not listed above is rejected. In particular, a lower-ranked observation (e.g. `confirmed` arriving after `finalized`) is never applied to the canonical receipt state.

### Out-of-order and duplicate events

- **Out-of-order events:** When an observation's status ranks lower than the receipt's current canonical status, the canonical state is left unchanged and the observation is recorded as conflict evidence (see below). A `receipt_status_conflict` event is emitted for downstream consumers.
- **Duplicate events:** An observation whose status equals the current canonical status is idempotent — no state change, no conflict record, and no event. Re-delivery of the same chain event is therefore safe.
- **Forward transitions:** Observations that advance the status along an allowed transition update the canonical state and are recorded as normal reconciliation history.

### Conflict evidence

Rejected observations are retained rather than dropped, so operators can audit late or duplicated chain events.

Table: `receipt_status_conflicts`

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | uuid | Primary key |
| `receiptId` | string | Receipt whose canonical state was protected |
| `txHash` | string | Transaction hash of the conflicting observation |
| `currentStatus` | string | Canonical status at the time of the observation |
| `observedStatus` | string | Status carried by the rejected observation |
| `source` | string | Origin of the observation (e.g. `chain-listener`, `reconciler`) |
| `observedAt` | timestamp | When the observation was produced |
| `createdAt` | timestamp | When the conflict was recorded |

Each recorded conflict also emits a `receipt_status_conflict` event with the same identifying fields so that alerting and dashboards can react without polling the table.

## Notification Delivery and Retry Semantics

This section documents how notification events flow from creation to delivery, what guarantees exist per channel, and where delivery status can lag. It is the integrator reference for anyone consuming notification events or building on top of `NotificationService`.

### Event creation flow

1. A domain action (e.g. `payment_received`, `recurring.payment.due`) emits an event via `@nestjs/event-emitter`.
2. A handler decorated with `@OnEvent('event_name', { async: true })` receives the event and calls `NotificationService` to create one or more notification records — one per recipient and channel.
3. Each notification record is persisted with its initial delivery status before any channel dispatch is attempted. Persistence is the source of truth; in-memory dispatch state is not.
4. The service resolves the recipient's notification preferences (see below) and enqueues a delivery attempt for each enabled channel.

Event emission is fire-and-forget from the emitter's perspective: the emitting domain action does not block on notification creation or delivery, and a notification failure never rolls back the originating domain action.

### Preferences handling

- Preferences are evaluated at notification creation time, per recipient and per channel.
- A channel that is disabled in the recipient's preferences produces no delivery attempt for that channel; the notification record is still created so the in-app history remains complete.
- Preference changes apply to notifications created after the change. Already-created notifications are not retroactively re-routed.

### Per-channel delivery guarantees

No channel provides an exactly-once guarantee. The strongest guarantee available is **at-least-once delivery with best-effort deduplication** (see below). Integrators must treat notification delivery as idempotent-tolerant.

| Channel | Delivery guarantee | Notes |
| :--- | :--- | :--- |
| In-app | At-least-once | Persisted record is the delivery; status reflects persistence, not user view. |
| Email | At-least-once | Provider acceptance is recorded; final inbox delivery is outside our control. |
| Webhook / push | At-least-once | Consumer endpoints must tolerate duplicate deliveries. |

Delivery status values reflect the last known outcome of a delivery attempt. A status of `delivered` means the channel accepted the notification, not that the end user has read or acted on it.

### Retry behavior and backoff

- Failed delivery attempts are retried with exponential backoff up to a bounded maximum number of attempts.
- Retries are per notification record and per channel; a failure on one channel does not block delivery on another.
- Once the maximum attempt count is exhausted, the notification is marked as permanently failed and no further automatic retries occur.
- Retries are not guaranteed to be immediate; backoff means delivery status can remain in a pending/retrying state for a period after the initial attempt.

### Deduplication and its idempotency boundary

- Deduplication is best-effort and keyed on the notification's identifying fields (recipient, event, and channel). It reduces duplicate deliveries caused by retries or re-emitted events.
- The idempotency boundary is the notification record's persistence layer: a duplicate that resolves to the same record does not create a second record.
- Deduplication does **not** guarantee exactly-once delivery. A retry after a partially successful attempt, or a re-emitted event that maps to a distinct record, can still result in a duplicate delivery. Consumers must be idempotent.

### When delivery status can be delayed

- **Backoff windows:** A notification awaiting a retry shows a pending/retrying status until the next attempt runs.
- **Async handlers:** Because handlers run with `{ async: true }`, status updates are not synchronous with the originating domain action.
- **Provider latency:** Email and webhook channels depend on external providers; status reflects provider acceptance, which can lag.
- **Terminal failures:** A notification that exhausts retries stays in a failed state and will not transition further without manual intervention.

## How to add new handlers
1. Inject `NotificationService` (or create a new service).
2. Use the `@OnEvent('event_name', { async: true })` decorator.
