# Test Plan

## Accounting — Unit / Integration Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Valid request, new idempotency key | `201`, delivery note created, key stored |
| 2 | Same request replayed (duplicate key) | `200`, same `delivery_note_id`, no second delivery note |
| 3 | Unknown `customer_ref` | `404`, `partner_not_found` error |
| 4 | Unknown `product_ref` | `404`, `product_not_found` error |
| 5 | Invalid unit of measure | `422`, `invalid_unit` error |
| 6 | Missing required fields | `400` |
| 7 | Missing `Idempotency-Key` header | `400` |
| 8 | Missing or invalid `Authorization` header | `401` |
| 9 | Two concurrent requests with the same key (race) | Exactly one `201`, one `200`; one delivery note |

## Sales Analyzer — Unit / Integration Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Order status changes to `shipped` | Outbox entry created with `status = pending` |
| 2 | Worker processes a `pending` entry, Accounting returns `201` | Entry marked `sent` |
| 3 | Worker processes entry, Accounting returns `500` | Entry stays `pending`, `attempts` incremented, `next_retry_at` set |
| 4 | Worker processes entry, Accounting returns `404` | Entry marked `failed_permanent` |
| 5 | Worker retries same entry multiple times | Same `Idempotency-Key` sent on every attempt |
| 6 | Outbox transaction: order status update fails after outbox insert | Outbox entry rolled back (no phantom event) |
| 7 | Outbox transaction: outbox insert fails after order update | Order update rolled back (no lost event) |

## End-to-End Tests (Staging)

| # | Scenario | Expected result |
|---|----------|-----------------|
| 1 | Ship an order in Sales Analyzer UI | Delivery note appears in Accounting within 1 minute |
| 2 | Ship the same order again (re-trigger) | Only one delivery note in Accounting |
| 3 | Disable Accounting temporarily, ship an order, re-enable | Delivery note eventually created after retries |
| 4 | Ship an order with an unknown customer | `failed_permanent` in outbox; alert raised; no delivery note |
| 5 | Fix the missing partner, re-trigger the outbox entry | Delivery note created successfully |

## Monitoring Checks

- No outbox entries stuck in `pending` for more than 2 hours without progression.
- No unexpected `4xx` errors from Accounting beyond known data-setup issues.
- Delivery note count in Accounting equals shipped order count in Sales Analyzer over any time window.
