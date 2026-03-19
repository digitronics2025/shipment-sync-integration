# Progress

Track implementation status here. Update as work completes.

## Accounting

- [ ] Migration: `integration_idempotency_keys` table
- [ ] Route: `POST /api/integrations/sales-analyzer/events`
- [ ] Idempotency check and storage (all event types)
- [ ] Partner resolution
- [ ] Product resolution
- [ ] `order_shipped`: delivery note creation
- [ ] `order_canceled_after_shipment`: receipt/reversal note creation (linked to original delivery)
- [ ] `order_returned`: receipt/return note creation (linked to original delivery)
- [ ] Auth middleware for integration endpoint
- [ ] Unit tests (see `docs/08-test-plan.md` — Accounting section)

## Sales Analyzer

- [ ] Migration: `shipment_outbox` table (with `event_type` and `canceled` status support)
- [ ] Hook: write outbox entry when order status → `shipped` (`event_type = order_shipped`)
- [ ] Hook: mark `order_shipped` outbox entry `canceled` when order canceled before sync
- [ ] Hook: write outbox entry when order canceled after sync (`event_type = order_canceled_after_shipment`)
- [ ] Hook: write outbox entry when order returned (`event_type = order_returned`)
- [ ] Background worker: poll and send pending entries (skip `canceled`), use correct idempotency key per event type
- [ ] Retry logic with backoff
- [ ] Permanent failure handling
- [ ] Env var: `ACCOUNTING_URL`
- [ ] Env var: `ACCOUNTING_INTEGRATION_TOKEN`
- [ ] Unit tests (see `docs/08-test-plan.md` — Sales Analyzer section)

## End-to-End

- [ ] Staging environment configured (partners + products set up)
- [ ] End-to-end scenarios validated (see `docs/08-test-plan.md` — E2E section)
- [ ] Rollout checklist completed (see `docs/07-rollout-plan.md`)

## Notes

- **Cancellation/return policy adopted** — see `docs/01-goal.md`. Post-shipment cancellations and returns produce explicit compensating documents (receipt/reversal or receipt/return notes). Posted documents are never modified or deleted by this integration.
