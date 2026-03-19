# Sales Analyzer Agent — Shipment Integration

You are working inside the **Sales Analyzer** repository only. Do not touch any other repository.

## Your task

Implement the outbox pattern and background worker that reliably sends order lifecycle events (shipped, canceled after shipment, returned) to the Accounting integration endpoint.

## What to implement

### 1. Database migration

Create a migration file that adds the `shipment_outbox` table:

```sql
CREATE TABLE shipment_outbox (
  id            SERIAL       PRIMARY KEY,
  order_id      INTEGER      NOT NULL REFERENCES orders(id),
  event_type    VARCHAR(40)  NOT NULL,
  payload       JSONB        NOT NULL,
  status        VARCHAR(20)  NOT NULL DEFAULT 'pending',
  attempts      INTEGER      NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP    NOT NULL DEFAULT NOW(),
  created_at    TIMESTAMP    NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE INDEX shipment_outbox_pending_idx
  ON shipment_outbox (status, next_retry_at)
  WHERE status = 'pending';
```

Supported `event_type` values: `order_shipped`, `order_canceled_after_shipment`, `order_returned`.

Supported `status` values: `pending`, `sent`, `failed_permanent`, `canceled`.

Provide the exact file path following this project's existing migration naming convention.

### 2. Outbox write on order shipped

In the existing code path where an order's status is set to `shipped`, add — within the **same database transaction** — an insert into `shipment_outbox`:

```json
{
  "event_type": "order_shipped",
  "order_id": "<order.id>",
  "shipped_at": "<shipped_at ISO 8601 UTC>",
  "customer_ref": "<order.customer_ref or equivalent>",
  "lines": [
    {
      "product_ref": "<line.product_ref or equivalent>",
      "quantity": "<line.quantity>",
      "unit": "<line.unit>"
    }
  ]
}
```

### 3. Outbox write on cancel before shipped sync

When an order is canceled and its `order_shipped` outbox entry is still `pending`:
- Mark the `order_shipped` entry as `canceled` (within the same transaction).
- Do **not** create an `order_canceled_after_shipment` entry — there is nothing to offset in Accounting.

### 4. Outbox write on cancel after shipped sync

When an order is canceled and its `order_shipped` outbox entry is already `sent`:
- Create a new outbox entry with `event_type = order_canceled_after_shipment`:

```json
{
  "event_type": "order_canceled_after_shipment",
  "order_id": "<order.id>",
  "canceled_at": "<canceled_at ISO 8601 UTC>",
  "reason": "<cancellation reason>",
  "original_delivery_note_id": "<delivery_note_id from the original shipped sync response, if stored>",
  "customer_ref": "<order.customer_ref>",
  "lines": [
    {
      "product_ref": "<line.product_ref>",
      "quantity": "<line.quantity>",
      "unit": "<line.unit>"
    }
  ]
}
```

### 5. Outbox write on return

When an order is returned and its `order_shipped` outbox entry is already `sent`:
- Create a new outbox entry with `event_type = order_returned`:

```json
{
  "event_type": "order_returned",
  "order_id": "<order.id>",
  "returned_at": "<returned_at ISO 8601 UTC>",
  "reason": "<return reason>",
  "original_delivery_note_id": "<delivery_note_id from the original shipped sync response, if stored>",
  "customer_ref": "<order.customer_ref>",
  "lines": [
    {
      "product_ref": "<line.product_ref>",
      "quantity": "<returned quantity>",
      "unit": "<line.unit>"
    }
  ]
}
```

### 6. Background worker

Create a background worker (following this project's existing worker/job conventions) that:

1. Runs every 30 seconds (or on a configurable schedule).
2. Queries: `SELECT * FROM shipment_outbox WHERE status = 'pending' AND next_retry_at <= NOW() ORDER BY id FOR UPDATE SKIP LOCKED`.
3. For each entry, sends a `POST` request to `ACCOUNTING_URL + /api/integrations/sales-analyzer/events` with:
   - `Content-Type: application/json`
   - `Authorization: Bearer <ACCOUNTING_INTEGRATION_TOKEN>`
   - `Idempotency-Key`: format depends on `event_type`:
     - `order_shipped` → `sales-analyzer:order-shipped:<order_id>`
     - `order_canceled_after_shipment` → `sales-analyzer:order-canceled:<order_id>`
     - `order_returned` → `sales-analyzer:order-returned:<order_id>`
   - Body: the stored `payload`
4. On `2xx`: update `status = 'sent'`, `updated_at = now()`. Store the response body (e.g. `delivery_note_id` or `receipt_note_id`) for future reference.
5. On `5xx` or network error: increment `attempts`, set `next_retry_at` using the retry schedule below, update `updated_at`.
6. On `4xx` (except `429`): set `status = 'failed_permanent'`, update `updated_at`. Log an alert.
7. On `429`: retry later using the `Retry-After` header value if present, otherwise use the standard schedule.

### Retry schedule

| attempts (after this failure) | next_retry_at delay |
|-------------------------------|---------------------|
| 1                             | 1 minute            |
| 2                             | 5 minutes           |
| 3                             | 15 minutes          |
| 4                             | 1 hour              |
| 5+                            | 4 hours             |

### 7. Environment variables

Add these to `.env.example` or equivalent:

```
ACCOUNTING_URL=https://accounting.internal
ACCOUNTING_INTEGRATION_TOKEN=<shared-secret>
```

### 8. Tests

Write tests covering all cases in `docs/08-test-plan.md` (Sales Analyzer section). Follow the testing conventions already used in this project.

## Assumptions to document

For each assumption you make about existing model names, column names, worker conventions, or file paths, write a short comment in the code or in a `## Assumptions` section in your reply.

## Output format

For every change, provide:
- Exact file path (relative to the repo root)
- Full file content or exact diff
- Migration SQL
- Any new env vars

Do not create Docker files, CI config, or package.json changes.

---

## CANCELLATION & RETURN POLICY — MUST FOLLOW

1. **Only `shipped` orders produce `order_shipped` outbox entries.** No other order state triggers a shipment event.
2. **Cancel before shipped sync:** If an order is canceled while its `order_shipped` outbox entry is still `pending`, mark that entry as `canceled`. The worker must **never send** a `canceled` entry. Do not create an `order_canceled_after_shipment` entry.
3. **Cancel after shipped sync:** If the `order_shipped` entry is already `sent`, create a new outbox entry with `event_type = order_canceled_after_shipment`. This triggers a receipt/reversal note in Accounting.
4. **Return after shipped sync:** If the `order_shipped` entry is already `sent`, create a new outbox entry with `event_type = order_returned`. This triggers a receipt/return note in Accounting.
5. **No silent deletions.** Never ask Accounting to modify or delete a posted document. Cancellations and returns are always represented by explicit compensating documents.
6. **Outbox `status` values:** `pending`, `sent`, `failed_permanent`, `canceled`.
7. **Idempotency keys are distinct per event type.** A single order can have up to three outbox entries (one per event type), each with a unique idempotency key.
