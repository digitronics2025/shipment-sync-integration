# Sales Analyzer Agent — Shipment Integration

You are working inside the **Sales Analyzer** repository only. Do not touch any other repository.

## Your task

Implement the outbox pattern and background worker that reliably sends shipment events to the Accounting integration endpoint.

## What to implement

### 1. Database migration

Create a migration file that adds the `shipment_outbox` table:

```sql
CREATE TABLE shipment_outbox (
  id            SERIAL       PRIMARY KEY,
  order_id      INTEGER      NOT NULL REFERENCES orders(id),
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

Provide the exact file path following this project's existing migration naming convention.

### 2. Outbox write on order shipped

In the existing code path where an order's status is set to `shipped`, add — within the **same database transaction** — an insert into `shipment_outbox`:

```json
{
  "order_id": <order.id>,
  "shipped_at": "<shipped_at ISO 8601 UTC>",
  "customer_ref": "<order.customer_ref or equivalent>",
  "lines": [
    {
      "product_ref": "<line.product_ref or equivalent>",
      "quantity": <line.quantity>,
      "unit": "<line.unit>"
    }
  ]
}
```

Provide the exact file path where this change should be made.

### 3. Background worker

Create a background worker (following this project's existing worker/job conventions) that:

1. Runs every 30 seconds (or on a configurable schedule).
2. Queries: `SELECT * FROM shipment_outbox WHERE status = 'pending' AND next_retry_at <= NOW() ORDER BY id FOR UPDATE SKIP LOCKED`.
3. For each entry, sends a `POST` request to `ACCOUNTING_URL + /api/integrations/sales-analyzer/shipments` with:
   - `Content-Type: application/json`
   - `Authorization: Bearer <ACCOUNTING_INTEGRATION_TOKEN>`
   - `Idempotency-Key: sales-analyzer:order-shipped:<order_id>`
   - Body: the stored `payload`
4. On `2xx`: update `status = 'sent'`, `updated_at = now()`.
5. On `5xx` or network error: increment `attempts`, set `next_retry_at` using the retry schedule below, update `updated_at`.
6. On `4xx` (except `429`): set `status = 'failed_permanent'`, update `updated_at`. Log an alert. This includes `409`, which means the same idempotency key was sent with a different payload — a code bug, not a transient error.
7. On `429`: retry later using the `Retry-After` header value if present, otherwise use the standard schedule.

### Retry schedule

| attempts (after this failure) | next_retry_at delay |
|-------------------------------|---------------------|
| 1                             | 1 minute            |
| 2                             | 5 minutes           |
| 3                             | 15 minutes          |
| 4                             | 1 hour              |
| 5+                            | 4 hours             |

### 4. Environment variables

Add these to `.env.example` or equivalent:

```
ACCOUNTING_URL=https://accounting.internal
ACCOUNTING_INTEGRATION_TOKEN=<shared-secret>
```

### 5. Tests

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
