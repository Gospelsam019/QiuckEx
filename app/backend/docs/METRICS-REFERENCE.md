# Metrics Reference

This document describes the metrics exposed by the backend and how they are
collected. It also documents the notification delivery and retry semantics that
integrators rely on when consuming notification-related metrics.

## Notification delivery and retry semantics

### Event creation

Notifications are created when a domain event is emitted (for example, a
payment status change or an account update). Each event is persisted before any
delivery attempt is made, so a notification record exists even if delivery is
delayed or fails. Event creation is the single source of truth for what should
be delivered; delivery workers only read from persisted events.

### Preferences

Per-user channel preferences are evaluated at delivery time, not at event
creation time. If a user disables a channel, pending deliveries for that channel
are skipped. Preference changes therefore affect notifications that have not yet
been delivered.

### Delivery guarantees per channel

Delivery is **at-least-once** per channel. There is no exactly-once guarantee:
retries and worker restarts can cause a notification to be delivered more than
once. Consumers that require exactly-once behavior must deduplicate using the
notification identifier carried with each delivery. The only idempotency
boundary is the persisted notification record keyed by its identifier; delivery
itself is not transactional with the downstream channel.

### Retries and backoff

Failed delivery attempts are retried with exponential backoff up to a bounded
number of attempts. After the maximum attempts are exhausted the notification is
marked as failed and no further automatic retries occur. Retry counters and
failure states are reflected in the notification metrics.

### Deduplication

Deduplication is applied at the event level: an event with the same identifier
is not re-created, so duplicate event emissions do not produce duplicate
notification records. This is the idempotency boundary. Deduplication does not
extend to delivery, which remains at-least-once as described above.

### Delayed delivery status

Delivery status can lag behind actual delivery because status updates are
written asynchronously by delivery workers. Metrics that report delivery status
may therefore be delayed relative to the underlying channel. Treat status
metrics as eventually consistent and rely on the notification identifier for
reconciliation.

## Metric collection

Metrics are aggregated from persisted events and delivery attempts. Because
status updates are asynchronous, counters may briefly lag behind the events they
describe.
