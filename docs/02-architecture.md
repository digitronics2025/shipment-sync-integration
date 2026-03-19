# Architecture

## Overview

```
Sales Analyzer                          Accounting
─────────────────                       ─────────────────────────────
Order → shipped
  │
  ▼
Outbox table (event_type = order_shipped)
  │
  ▼ (background worker, retries)
POST /api/integrations/sales-analyzer/events  ──►  Integration endpoint
                                                         │
                                                         ▼
                                                  Idempotency check
                                                         │
                                                         ▼
                                                  Partner resolution
                                                         │
                                                         ▼
                                                  Product resolution
                                                         │
                                                         ▼
                                                  Create delivery note

Order → canceled (after shipment)
  │
  ▼
Outbox table (event_type = order_canceled_after_shipment)
  │
  ▼ (background worker, retries)
POST /api/integrations/sales-analyzer/events  ──►  Integration endpoint
                                                         │
                                                         ▼
                                                  Idempotency check
                                                         │
                                                         ▼
                                                  Lookup original delivery
                                                         │
                                                         ▼
                                                  Create receipt/reversal note

Order → returned
  │
  ▼
Outbox table (event_type = order_returned)
  │
  ▼ (background worker, retries)
POST /api/integrations/sales-analyzer/events  ──►  Integration endpoint
                                                         │
                                                         ▼
                                                  Idempotency check
                                                         │
                                                         ▼
                                                  Lookup original delivery
                                                         │
                                                         ▼
                                                  Create receipt/return note
```

## Responsibilities

### Sales Analyzer

- Detect when an order status changes to `shipped`, `canceled` (after shipment), or `returned`.
- Write an event to a local **outbox table** within the same database transaction.
- Run a background worker that reads pending outbox entries and calls the Accounting endpoint.
- Mark entries as `sent` after a `2xx` response, or record the failure and retry with backoff.
- If an order is canceled before its shipment outbox entry is sent, mark that entry `canceled` — never send it.

### Accounting

- Expose `POST /api/integrations/sales-analyzer/events`.
- Enforce idempotency using the `Idempotency-Key` header.
- For `order_shipped`: resolve partner and products, create a delivery note.
- For `order_canceled_after_shipment`: look up the original delivery note, create a receipt/reversal note that offsets it.
- For `order_returned`: look up the original delivery note, create a receipt/return note that offsets it.
- Return a stable response for duplicate requests (same key → same result, no second write).

## Design Principles

- **Accounting is the source of truth** for inventory and accounting entries.
- **No fuzzy matching**: product and partner resolution must be exact; unknown items are rejected with a clear error.
- **Retries must never create duplicates**: idempotency is enforced server-side in Accounting.
- **Keep coupling minimal**: Sales Analyzer only needs to know the endpoint URL and the payload shape.
- **No silent deletions**: posted documents in Accounting are never modified or deleted by this integration. Cancellations and returns produce explicit compensating documents (receipt/reversal or receipt/return notes).
- **Compensating documents link to originals**: every receipt/reversal or receipt/return note must reference the original delivery note it offsets.
