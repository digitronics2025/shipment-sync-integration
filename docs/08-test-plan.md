# Test Plan

## Accounting — Unit / Integration Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Valid `order_shipped` request, new idempotency key | `201`, delivery note created, key stored |
| 2 | Same `order_shipped` request replayed (duplicate key) | `200`, same `delivery_note_id`, no second delivery note |
| 3 | Unknown `customer_ref` | `404`, `partner_not_found` error |
| 4 | Unknown `product_ref` | `404`, `product_not_found` error |
| 5 | Invalid unit of measure | `422`, `invalid_unit` error |
| 6 | Missing required fields | `400` |
| 7 | Missing `Idempotency-Key` header | `400` |
| 8 | Missing or invalid `Authorization` header | `401` |
| 9 | Two concurrent requests with the same key (race) | Exactly one `201`, one `200`; one document |
| 10 | Unknown `event_type` | `400`, `invalid_event_type` error |
| 11 | Valid `order_canceled_after_shipment`, original delivery exists | `201`, receipt/reversal note created referencing original delivery |
| 12 | Same `order_canceled_after_shipment` replayed (duplicate key) | `200`, same `receipt_note_id`, no second receipt note |
| 13 | `order_canceled_after_shipment` but no original delivery found | `404`, `delivery_not_found` error |
| 14 | Valid `order_returned`, original delivery exists | `201`, receipt/return note created referencing original delivery |
| 15 | Same `order_returned` replayed (duplicate key) | `200`, same `receipt_note_id`, no second receipt note |
| 16 | `order_returned` but no original delivery found | `404`, `delivery_not_found` error |
| 17 | `order_returned` with quantity exceeding original delivery | `422`, `quantity_exceeds_original` error |

## Sales Analyzer — Unit / Integration Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Order status changes to `shipped` | Outbox entry created with `event_type = order_shipped`, `status = pending` |
| 2 | Worker processes a `pending` entry, Accounting returns `201` | Entry marked `sent` |
| 3 | Worker processes entry, Accounting returns `500` | Entry stays `pending`, `attempts` incremented, `next_retry_at` set |
| 4 | Worker processes entry, Accounting returns `404` | Entry marked `failed_permanent` |
| 5 | Worker retries same entry multiple times | Same `Idempotency-Key` sent on every attempt |
| 6 | Outbox transaction: order status update fails after outbox insert | Outbox entry rolled back (no phantom event) |
| 7 | Outbox transaction: outbox insert fails after order update | Order update rolled back (no lost event) |
| 8 | Order canceled while `order_shipped` outbox entry is `pending` | Entry marked `canceled`, worker never sends it; no `order_canceled_after_shipment` entry created |
| 9 | Worker picks up entries: mix of `pending` and `canceled` | Only `pending` entries are processed; `canceled` entries are skipped |
| 10 | Order canceled after `order_shipped` entry is `sent` | New outbox entry created with `event_type = order_canceled_after_shipment`, `status = pending` |
| 11 | Order returned after `order_shipped` entry is `sent` | New outbox entry created with `event_type = order_returned`, `status = pending` |
| 12 | Worker sends `order_canceled_after_shipment`, Accounting returns `201` | Entry marked `sent` |
| 13 | Worker sends `order_returned`, Accounting returns `201` | Entry marked `sent` |
| 14 | Correct `Idempotency-Key` format used per event type | `order_shipped` → `sales-analyzer:order-shipped:<id>`, `order_canceled_after_shipment` → `sales-analyzer:order-canceled:<id>`, `order_returned` → `sales-analyzer:order-returned:<id>` |

## End-to-End Tests (Staging)

| # | Scenario | Expected result |
|---|----------|-----------------|
| 1 | Ship an order in Sales Analyzer UI | Delivery note appears in Accounting within 1 minute |
| 2 | Ship the same order again (re-trigger) | Only one delivery note in Accounting |
| 3 | Disable Accounting temporarily, ship an order, re-enable | Delivery note eventually created after retries |
| 4 | Ship an order with an unknown customer | `failed_permanent` in outbox; alert raised; no delivery note |
| 5 | Fix the missing partner, re-trigger the outbox entry | Delivery note created successfully |
| 6 | Cancel an order before shipment sync completes (outbox still `pending`) | Outbox entry marked `canceled`; no delivery note in Accounting; no cancel event created |
| 7 | Cancel an order after shipment sync completes (outbox is `sent`) | Receipt/reversal note created in Accounting referencing the original delivery; original delivery unchanged |
| 8 | Return an order after shipment sync completes | Receipt/return note created in Accounting referencing the original delivery; original delivery unchanged |
| 9 | Return a partial quantity after shipment sync | Receipt/return note created with partial quantity; original delivery unchanged |
| 10 | Cancel after shipment, then replay the cancel event | Only one receipt/reversal note in Accounting (idempotency) |

## Monitoring Checks

- No outbox entries stuck in `pending` for more than 2 hours without progression.
- No unexpected `4xx` errors from Accounting beyond known data-setup issues.
- Delivery note count in Accounting equals shipped order count in Sales Analyzer over any time window.
- Every `order_canceled_after_shipment` event has a matching receipt/reversal note in Accounting.
- Every `order_returned` event has a matching receipt/return note in Accounting.
