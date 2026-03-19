# Idempotency and Retries

## Idempotency (Accounting side)

### Key format per event type

| Event type                       | Idempotency-Key format                        |
|----------------------------------|-----------------------------------------------|
| `order_shipped`                  | `sales-analyzer:order-shipped:<order_id>`     |
| `order_canceled_after_shipment`  | `sales-analyzer:order-canceled:<order_id>`    |
| `order_returned`                 | `sales-analyzer:order-returned:<order_id>`    |

Each event type for a given order uses a **distinct** idempotency key. A single order can have up to three keys (one per event type).

### Storage

Accounting stores every processed `Idempotency-Key` in a dedicated table (e.g. `integration_idempotency_keys`) with:

| Column          | Type        | Notes                              |
|-----------------|-------------|------------------------------------|
| `key`           | string (PK) | The full idempotency key           |
| `response_code` | integer     | HTTP status returned (200 or 201)  |
| `response_body` | jsonb       | Full response body stored as JSON  |
| `created_at`    | timestamp   |                                    |

### Behaviour

1. On every incoming request, Accounting first checks whether the key already exists.
2. If it exists, return the stored `response_code` and `response_body` immediately — **no business logic is re-executed**.
3. If it does not exist, process the request normally and persist the key + response atomically with the document creation.
4. The check and insert must be done inside a **database transaction** with a unique constraint on `key` to prevent race conditions.

### Expiry

Idempotency keys are kept indefinitely (or for a configurable retention period, e.g. 90 days). There is no need to expire them in the short term.

---

## Retries (Sales Analyzer side)

### Outbox table

Sales Analyzer maintains an outbox table (e.g. `shipment_outbox`) with:

| Column          | Type      | Notes                                                       |
|-----------------|-----------|-------------------------------------------------------------|
| `id`            | integer   | Primary key                                                 |
| `order_id`      | integer   | Foreign key to orders                                       |
| `event_type`    | string    | `order_shipped`, `order_canceled_after_shipment`, `order_returned` |
| `payload`       | jsonb     | Full request body to send                                   |
| `status`        | string    | `pending`, `sent`, `failed_permanent`, `canceled`           |
| `attempts`      | integer   | Number of send attempts so far                              |
| `next_retry_at` | timestamp | When to attempt next (null if `sent`)                       |
| `created_at`    | timestamp |                                                             |
| `updated_at`    | timestamp |                                                             |

### Retry schedule

| Attempt | Delay before next retry |
|---------|------------------------|
| 1       | 1 minute               |
| 2       | 5 minutes              |
| 3       | 15 minutes             |
| 4       | 1 hour                 |
| 5+      | 4 hours                |

### Status transitions

- `pending` → `sent`: Accounting returned `2xx`.
- `pending` → `pending` (increment `attempts`, set `next_retry_at`): Accounting returned `5xx` or network error.
- `pending` → `failed_permanent`: Accounting returned `4xx` (except `429`). This includes `409`, which signals the same idempotency key was sent with a different payload — a bug that requires investigation, not a retry.
- `pending` → `pending` (retry later): Accounting returned `429` — use `Retry-After` header if provided.
- `pending` → `canceled`: The order was canceled before the `order_shipped` outbox entry was successfully sent. The entry must not be retried.

### Worker behaviour

- Runs on a schedule (e.g. every 30 seconds).
- Picks up all entries where `status = 'pending'` and `next_retry_at <= now()`.
- Sends each entry with the correct `Idempotency-Key` header on every attempt (format depends on `event_type`).
- Never spawns parallel workers for the same `order_id`.

### Key guarantees

- Because Sales Analyzer always sends the same `Idempotency-Key` for a given `order_id` + `event_type`, and Accounting enforces idempotency server-side, **retries can never create duplicate documents**.
- Each event type uses a distinct key prefix, so a shipment and its subsequent cancellation/return are independent idempotent operations.

---

## Cancellation & Return Rules

- The outbox worker must **never send** an entry whose status is `canceled`.
- If an order is canceled while its `order_shipped` outbox entry is still `pending`, that entry must be set to `canceled` before the next worker run. No `order_canceled_after_shipment` event is created in this case (there is nothing to offset in Accounting).
- If the `order_shipped` entry has already been sent and a delivery note exists in Accounting, a new outbox entry with `event_type = order_canceled_after_shipment` is created to trigger a receipt/reversal note.
- Returns follow the same pattern: a new outbox entry with `event_type = order_returned` triggers a receipt/return note.
- Posted documents in Accounting are **never modified or deleted** by this integration. Every post-shipment cancellation or return is represented by an explicit compensating document.
