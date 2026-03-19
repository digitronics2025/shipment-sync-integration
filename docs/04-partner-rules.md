# Partner Resolution Rules

## Responsibility

Accounting is responsible for resolving the partner (customer) from the `customer_ref` field in the request payload.

## Resolution Logic

1. Look up the partner in Accounting's partner table using `customer_ref` as an **exact match** against the partner's external reference field (e.g. `external_ref` or equivalent column).
2. If exactly one partner is found, use it.
3. If no partner is found, return `404` with a clear error message:
   ```json
   { "error": "partner_not_found", "customer_ref": "CUST-007" }
   ```
4. If multiple partners match (should not happen if `external_ref` is unique), return `409` and alert the team.

## No Fuzzy Matching

- Do **not** attempt name-based or partial matching.
- Do **not** auto-create partners.
- The `customer_ref` value must be set up in Accounting before the integration can process that customer's orders.

## Setup Requirement

Before go-live, every customer in Sales Analyzer that will generate shipments must have a corresponding partner in Accounting with a matching `external_ref`.

## Applicability

Partner resolution applies to all event types:

- `order_shipped`: resolve partner to create the delivery note.
- `order_canceled_after_shipment` and `order_returned`: the `customer_ref` must match the same partner as the original shipment. Accounting should validate consistency.

## Error Handling

- Sales Analyzer treats `404` from Accounting as a **permanent failure** (no retry).
- The outbox entry is marked `failed_permanent` and an alert is raised.
- Operations must manually create the partner and re-trigger the event.
