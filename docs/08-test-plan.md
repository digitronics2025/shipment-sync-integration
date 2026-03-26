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

## Sales Analyzer — Outbox UI Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Open outbox page with no filters | All entries listed, most recent first |
| 2 | Filter by `status = failed_permanent` | Only failed entries shown |
| 3 | Filter by `event_type = order_shipped` | Only shipped entries shown |
| 4 | Filter by `order_id` | Only entries for that order shown |
| 5 | View entry detail | Payload, status, attempts, next retry, timestamps, Accounting response displayed |
| 6 | Retry a `failed_permanent` entry | Status reset to `pending`, `next_retry_at` set to now; worker picks it up on next run |
| 7 | Cancel a `pending` entry | Status set to `canceled`; worker never sends it |
| 8 | Summary counts displayed | Correct totals for pending, sent, failed, canceled per event type |

## Accounting — Integration Audit UI Tests

| # | Test case | Expected result |
|---|-----------|-----------------|
| 1 | Open audit page with no filters | All received events listed, most recent first |
| 2 | Filter by `event_type = order_shipped` | Only shipped events shown |
| 3 | Filter by `status = failed` | Only failed events shown |
| 4 | Filter by `external_order_id` | Only events for that order shown |
| 5 | View event detail | Payload, idempotency key, status, error info, linked document displayed |
| 6 | Click linked document (delivery note) | Navigates to the delivery note in Accounting UI |
| 7 | Click linked document (receipt/reversal note) | Navigates to the receipt note; shows original delivery reference |
| 8 | Summary counts displayed | Correct totals for created, replayed, failed per event type |
| 9 | Verify page is read-only | No edit, delete, or retry actions available |

## UI Validation Checklist

Run through these steps on staging with both UI pages open side by side.

### Shipped — happy path

- [ ] Ship an order in Sales Analyzer → outbox page shows `order_shipped` entry with `status = pending`
- [ ] Worker sends it → outbox status updates to `sent`; Accounting audit page shows the event with `status = created` and a clickable delivery note link

### Cancel before send

- [ ] Ship an order, then click **Cancel** on the `pending` outbox entry before the worker runs → status changes to `canceled`
- [ ] Confirm the worker skips it (status stays `canceled`) and no event appears on the Accounting audit page

### Cancel after shipped sync

- [ ] Ship an order, wait for `sent`, then cancel the order → new `order_canceled_after_shipment` entry appears in the outbox as `pending`
- [ ] Worker sends it → outbox status updates to `sent`; Accounting audit page shows a receipt/reversal event linked to the original delivery note
- [ ] Open the receipt/reversal detail on the Accounting page → original delivery reference is present and clickable

### Return after shipped sync

- [ ] Ship an order, wait for `sent`, then return the order → new `order_returned` entry appears in the outbox as `pending`
- [ ] Worker sends it → outbox status updates to `sent`; Accounting audit page shows a receipt/return event linked to the original delivery note
- [ ] Open the receipt/return detail on the Accounting page → original delivery reference is present and clickable

### Retry safety

- [ ] Ship an order with an unknown product → outbox entry becomes `failed_permanent`
- [ ] Fix the product in Accounting, then click **Retry now** on the outbox entry → status resets to `pending`, worker picks it up
- [ ] Delivery note created in Accounting; only one document exists (no duplicates from the retry)

### Accounting page is read-only

- [ ] Confirm no edit, delete, retry, or cancel buttons exist on the Accounting audit page
- [ ] Confirm event detail view has no write actions

### Mobile / small-screen basics

- [ ] Both pages render without horizontal scroll on a 375px-wide viewport
- [ ] Filters and summary counts are usable on mobile
- [ ] Entry detail view is readable without truncation of key fields (order ID, status, event type)

## E2E Validation Checklist

Run this checklist against staging before rollout. Every item must pass.

### Prerequisites

- [ ] Staging Accounting and Sales Analyzer deployed with latest code
- [ ] At least one partner in Accounting with matching `external_ref`
- [ ] At least one product in Accounting with matching SKU
- [ ] Shared auth token configured in both apps
- [ ] `ACCOUNTING_URL` set in Sales Analyzer
- [ ] Background worker enabled in Sales Analyzer

### Shipped flow

- [ ] Ship an order → delivery note appears in Accounting
- [ ] Ship the same order again → no duplicate delivery note (idempotency)
- [ ] Verify `Idempotency-Key` format: `sales-analyzer:order-shipped:<order_id>`

### Cancel before sync

- [ ] Ship an order, immediately cancel before worker sends → outbox entry marked `canceled`, no delivery note in Accounting

### Cancel after sync

- [ ] Ship an order, wait for delivery note, then cancel → receipt/reversal note created in Accounting referencing original delivery
- [ ] Replay the cancel event → no duplicate receipt/reversal note (idempotency)
- [ ] Verify original delivery note is unchanged

### Return after sync

- [ ] Ship an order, wait for delivery note, then return full quantity → receipt/return note created referencing original delivery
- [ ] Return partial quantity → receipt/return note with partial quantity; original delivery unchanged
- [ ] Replay the return event → no duplicate receipt/return note (idempotency)

### Error handling

- [ ] Ship with unknown `customer_ref` → `failed_permanent` in outbox, no delivery note
- [ ] Ship with unknown `product_ref` → `failed_permanent` in outbox, no delivery note
- [ ] Simulate Accounting `500` → worker retries with backoff, delivery note eventually created

### Data integrity

- [ ] Delivery note count in Accounting matches shipped order count in Sales Analyzer
- [ ] Every cancel-after-sync has a matching receipt/reversal note
- [ ] Every return has a matching receipt/return note
- [ ] No outbox entries stuck in `pending` for more than 2 hours

## Monitoring Checks

- No outbox entries stuck in `pending` for more than 2 hours without progression.
- No unexpected `4xx` errors from Accounting beyond known data-setup issues.
- Delivery note count in Accounting equals shipped order count in Sales Analyzer over any time window.
- Every `order_canceled_after_shipment` event has a matching receipt/reversal note in Accounting.
- Every `order_returned` event has a matching receipt/return note in Accounting.
