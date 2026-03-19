# Goal

## Business Objective

When an order in **Sales Analyzer** transitions through key lifecycle states — shipped, canceled after shipment, or returned — the corresponding accounting documents must be created in **Accounting**.

## Scope

- Sales Analyzer emits events for orders that become `shipped`, `canceled` (after shipment), or `returned`.
- Accounting receives those events, resolves business rules, and creates the appropriate document.
- This integration does **not** cover inventory sync (handled separately).

## Supported Event Types

| Event | Trigger | Accounting action |
|-------|---------|-------------------|
| `order_shipped` | Order status → `shipped` | Create a **delivery note** |
| `order_canceled_after_shipment` | Order canceled after delivery note exists | Create a **receipt/reversal note** that offsets the original delivery |
| `order_returned` | Order returned after delivery note exists | Create a **receipt/return note** that offsets the original delivery |

## Cancellation & Return Policy

- **Only shipped orders create a delivery note in Accounting.** No other order state triggers an initial delivery.
- **Cancel before shipped sync:** If an order is canceled before the shipment event is sent to Accounting, no Accounting action is taken. Any pending shipment outbox entry in Sales Analyzer must be marked `canceled` and never retried.
- **Cancel after shipped sync:** A receipt/reversal note is created in Accounting that offsets the original delivery. The original delivery note is **never modified or deleted**.
- **Return after shipped sync:** A receipt/return note is created in Accounting that offsets the original delivery. The original delivery note is **never modified or deleted**.
- **No silent deletions:** Posted documents in Accounting are never deleted by this integration. Every post-shipment cancellation or return is represented by an explicit compensating document.

## Out of Scope

- Inventory adjustments triggered by shipments.
- Historical backfill of past shipments.

## Success Criteria

1. Every shipped order in Sales Analyzer has a matching delivery note in Accounting.
2. Every post-shipment cancellation has a matching receipt/reversal note in Accounting.
3. Every return has a matching receipt/return note in Accounting.
4. No duplicate documents are created, even if events are retried.
5. Compensating documents reference the original delivery they offset.
6. Failures are retried automatically and do not require manual intervention for transient errors.
