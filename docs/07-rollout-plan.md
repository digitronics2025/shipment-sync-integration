# Rollout Plan

## Phases

### Phase 1 — Implementation (no live traffic)

1. **Accounting** implements the integration endpoint for all event types and idempotency logic (see `prompts/accounting-agent.md`).
2. **Sales Analyzer** implements the outbox table, background worker, and event emission for shipped/canceled/returned (see `prompts/sales-analyzer-agent.md`).
3. Both sides are deployed behind a feature flag or with the worker **disabled**.

### Phase 2 — Internal testing

1. Run the test scenarios in `docs/08-test-plan.md` against a staging environment.
2. Verify idempotency: replay the same event twice, confirm only one document is created.
3. Verify retry: simulate a `500` from Accounting, confirm the worker retries.
4. Verify permanent failure: send an unknown `customer_ref`, confirm the outbox entry is marked `failed_permanent`.
5. Verify cancellation before sync: cancel an order with a `pending` outbox entry, confirm it is marked `canceled` and never sent.
6. Verify cancellation after sync: cancel a shipped order, confirm a receipt/reversal note is created in Accounting referencing the original delivery.
7. Verify return: return a shipped order, confirm a receipt/return note is created in Accounting referencing the original delivery.
8. Verify Sales Analyzer outbox UI: list, filter, retry, and cancel actions work correctly.
9. Verify Accounting audit UI: events appear with correct status, linked documents are clickable.

### Phase 3 — Controlled rollout

1. Enable the worker for a small subset of orders (e.g. by warehouse or product category).
2. Monitor Accounting for unexpected `404` or `422` errors.
3. Confirm delivery notes, receipt/reversal notes, and receipt/return notes appear correctly.

### Phase 4 — Full rollout

1. Enable the worker for all orders.
2. Run the end-to-end test suite one final time.
3. Remove the feature flag.

## Rollback

- Disable the background worker in Sales Analyzer (no code change required if feature-flagged).
- Accounting's endpoint can be disabled without affecting any other functionality.
- No data is lost: outbox entries remain `pending` and can be re-processed after the issue is fixed.

## Prerequisites Before Go-Live

- [ ] All customers in Sales Analyzer have a matching partner in Accounting (`external_ref` set).
- [ ] All product SKUs used in Sales Analyzer exist in Accounting.
- [ ] The shared auth token is configured in both apps via environment variables.
- [ ] Accounting's URL is set in Sales Analyzer's environment variables.
- [ ] Database migrations have run in both apps.
- [ ] Confirm Sales Analyzer marks `order_shipped` outbox entries as `canceled` when an order is canceled before sync completes.
- [ ] Confirm Sales Analyzer creates `order_canceled_after_shipment` outbox entries when an order is canceled after sync completes.
- [ ] Confirm Sales Analyzer creates `order_returned` outbox entries when an order is returned after sync completes.
- [ ] Confirm Accounting creates receipt/reversal and receipt/return notes that reference the original delivery.
- [ ] Sales Analyzer outbox UI deployed and accessible to operators.
- [ ] Accounting integration audit UI deployed and accessible to operators.
