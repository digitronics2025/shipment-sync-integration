# Architecture

## Overview

```
Sales Analyzer                          Accounting
─────────────────                       ─────────────────────────────
Order → shipped
  │
  ▼
Outbox table
  │
  ▼ (background worker, retries)
POST /api/integrations/sales-analyzer/shipments  ──►  Integration endpoint
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
```

## Responsibilities

### Sales Analyzer

- Detect when an order status changes to `shipped`.
- Write an event to a local **outbox table** within the same database transaction.
- Run a background worker that reads pending outbox entries and calls the Accounting endpoint.
- Mark entries as `sent` after a `2xx` response, or record the failure and retry with backoff.

### Accounting

- Expose `POST /api/integrations/sales-analyzer/shipments`.
- Enforce idempotency using the `Idempotency-Key` header.
- Resolve the partner (customer) from the payload.
- Resolve each line item's product from the payload.
- Create the delivery note using existing internal logic.
- Return a stable response for duplicate requests (same key → same result, no second write).

## Design Principles

- **Accounting is the source of truth** for inventory and accounting entries.
- **No fuzzy matching**: product and partner resolution must be exact; unknown items are rejected with a clear error.
- **Retries must never create duplicates**: idempotency is enforced server-side in Accounting.
- **Keep coupling minimal**: Sales Analyzer only needs to know the endpoint URL and the payload shape.
