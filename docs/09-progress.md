# Progress

Track implementation status here. Update as work completes.

## Accounting

- [x] Migration: `integration_shipments` table
- [x] Route: `POST /api/integrations/sales-analyzer/events`
- [x] Idempotency check and storage (all event types)
- [x] Partner resolution
- [x] Product resolution
- [x] `order_shipped`: delivery note creation
- [x] `order_canceled_after_shipment`: receipt/reversal note creation (linked to original delivery)
- [x] `order_returned`: receipt/return note creation (linked to original delivery)
- [x] Auth middleware for integration endpoint
- [x] UUID `order_id` mismatch resolved (both sides now use consistent format)
- [ ] Unit tests (see `docs/08-test-plan.md` — Accounting section)
- [ ] Integration audit UI: list received events with filters (`event_type`, `status`, `external_order_id`, date range)
- [ ] Integration audit UI: event detail view with linked document click-through
- [ ] Integration audit UI: summary counts (created, replayed, failed per event type)

## Sales Analyzer

- [x] Migration: `shipment_outbox` table (with `event_type` and `canceled` status support)
- [x] Hook: write outbox entry when order status → `shipped` (`event_type = order_shipped`)
- [x] Hook: mark `order_shipped` outbox entry `canceled` when order canceled before sync
- [x] Hook: write outbox entry when order canceled after sync (`event_type = order_canceled_after_shipment`)
- [x] Hook: write outbox entry when order returned (`event_type = order_returned`)
- [x] Background worker: poll and send pending entries (skip `canceled`), use correct idempotency key per event type
- [x] Retry logic with backoff
- [x] Permanent failure handling
- [x] Env var: `ACCOUNTING_URL`
- [x] Env var: `ACCOUNTING_INTEGRATION_TOKEN`
- [ ] Unit tests (see `docs/08-test-plan.md` — Sales Analyzer section)
- [ ] Outbox control UI: list entries with filters (`status`, `event_type`, `order_id`, date range)
- [ ] Outbox control UI: entry detail view (payload, status, attempts, Accounting response)
- [ ] Outbox control UI: manual retry action (`failed_permanent` → `pending`)
- [ ] Outbox control UI: manual cancel action (`pending` → `canceled`)
- [ ] Outbox control UI: summary counts (pending, sent, failed, canceled per event type)

## End-to-End

- [ ] Staging environment configured (partners + products set up)
- [ ] E2E validation checklist completed (see `docs/08-test-plan.md` — E2E Validation Checklist)
- [ ] End-to-end scenarios validated (see `docs/08-test-plan.md` — E2E section)
- [ ] Rollout checklist completed (see `docs/07-rollout-plan.md`)

## Notes

- **Both implementations aligned** — Accounting and Sales Analyzer repos implement the same contract as of 2026-03-19.
- **UUID order_id mismatch resolved** — both sides now use a consistent `order_id` format.
- **All three event types supported** — `order_shipped`, `order_canceled_after_shipment`, `order_returned`.
- **Cancellation/return policy adopted** — see `docs/01-goal.md`. Posted documents are never modified or deleted; corrections are explicit compensating documents.
- **Next step** — run the E2E validation checklist in `docs/08-test-plan.md` against staging.
