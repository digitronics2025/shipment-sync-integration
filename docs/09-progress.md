# Progress

Track implementation status here. Update as work completes.

## Accounting

- [ ] Migration: `integration_idempotency_keys` table
- [ ] Route: `POST /api/integrations/sales-analyzer/shipments`
- [ ] Idempotency check and storage
- [ ] Partner resolution
- [ ] Product resolution
- [ ] Delivery note creation
- [ ] Auth middleware for integration endpoint
- [ ] Unit tests (see `docs/08-test-plan.md` — Accounting section)

## Sales Analyzer

- [ ] Migration: `shipment_outbox` table
- [ ] Hook: write outbox entry when order status → `shipped`
- [ ] Background worker: poll and send pending entries
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

_Add notes, decisions, and blockers here as the work progresses._
