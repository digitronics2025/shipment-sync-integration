# Rollout Plan

## Phases

### Phase 1 — Implementation (no live traffic)

1. **Accounting** implements the integration endpoint and idempotency logic (see `prompts/accounting-agent.md`).
2. **Sales Analyzer** implements the outbox table and background worker (see `prompts/sales-analyzer-agent.md`).
3. Both sides are deployed behind a feature flag or with the worker **disabled**.

### Phase 2 — Internal testing

1. Run the test scenarios in `docs/08-test-plan.md` against a staging environment.
2. Verify idempotency: replay the same event twice, confirm only one delivery note is created.
3. Verify retry: simulate a `500` from Accounting, confirm the worker retries.
4. Verify permanent failure: send an unknown `customer_ref`, confirm the outbox entry is marked `failed_permanent`.

### Phase 3 — Controlled rollout

1. Enable the worker for a small subset of orders (e.g. by warehouse or product category).
2. Monitor Accounting for unexpected `404` or `422` errors.
3. Confirm delivery notes appear correctly.

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
