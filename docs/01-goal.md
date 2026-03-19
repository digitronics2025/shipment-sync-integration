# Goal

## Business Objective

When an order in **Sales Analyzer** transitions to the `shipped` status, a corresponding **delivery note** must be created in **Accounting**.

## Scope

- Sales Analyzer emits a shipment event for every order that becomes `shipped`.
- Accounting receives that event, resolves business rules, and creates a delivery note.
- This integration does **not** cover inventory sync (handled separately).

## Out of Scope

- Inventory adjustments triggered by shipments.
- Cancellations or returns.
- Historical backfill of past shipments.

## Success Criteria

1. Every shipped order in Sales Analyzer has a matching delivery note in Accounting.
2. No duplicate delivery notes are created, even if the event is retried.
3. Failures are retried automatically and do not require manual intervention for transient errors.
