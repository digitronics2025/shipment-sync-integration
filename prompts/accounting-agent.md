You are the accounting-side implementation agent for the shipment-to-delivery integration.

IMPORTANT:
You must strictly follow the contract and rules defined in the coordination repo:
- docs/03-api-contract.md
- docs/04-partner-rules.md
- docs/05-product-rules.md
- docs/06-idempotency-retries.md

You are working ONLY inside the Accounting repository:
tenten-accounting-in-main

---

## GOAL

Implement a single integration endpoint that receives events from Sales Analyzer and creates the appropriate accounting document.

Endpoint:
POST /api/integrations/sales-analyzer/events

Supported event types:
- `order_shipped` → create a delivery note
- `order_canceled_after_shipment` → create a receipt/reversal note that offsets the original delivery
- `order_returned` → create a receipt/return note that offsets the original delivery

---

## REQUIRED FEATURES

### 1. New DB table (integration receipts)

Create a new table (name: `integration_shipments`):

Columns:
- id (pk)
- source (string)
- event_type (string: order_shipped | order_canceled_after_shipment | order_returned)
- external_order_id (string)
- external_order_number (string)
- idempotency_key (string)
- request_hash (string)
- payload_json (json)
- delivery_id (string or uuid, nullable — set for order_shipped)
- delivery_no (string, nullable — set for order_shipped)
- receipt_note_id (string or uuid, nullable — set for cancel/return)
- receipt_note_no (string, nullable — set for cancel/return)
- original_delivery_id (string or uuid, nullable — set for cancel/return, links to the original delivery)
- status (string: created | replayed | failed)
- error_code (string nullable)
- error_message (string nullable)
- created_at
- updated_at

Constraints:
- UNIQUE (source, external_order_id, event_type)

---

### 2. Idempotency

- Read `Idempotency-Key` header
- Key format depends on event type (see docs/06-idempotency-retries.md)
- If same key already processed:
  - return previous result (replayed=true)
- If same external_order_id + event_type already exists:
  - return existing document (no duplicate creation)

---

### 3. Endpoint implementation

Create:
functions/api/integrations/sales-analyzer/events.ts

Responsibilities:

- authenticate request (reuse existing auth pattern)
- validate payload strictly
- validate required fields:
  - event_type
  - source
  - externalOrderId
  - lines
- route to the correct handler based on `event_type`
- enforce product rules
- enforce partner rules

---

### 4. Event: `order_shipped`

- Resolve partner from `customer_ref`
- Resolve products from `lines[].product_ref`
- Create delivery note using existing logic
- Return `{ "delivery_note_id": ... }`

---

### 5. Event: `order_canceled_after_shipment`

- Look up the original delivery note by `original_delivery_note_id` or by `order_id`
- If original delivery not found → return `404` with `delivery_not_found` error
- Resolve partner and products (same rules as shipment)
- Create a receipt/reversal note that offsets the original delivery
- The receipt/reversal note must reference the original delivery
- Return `{ "receipt_note_id": ..., "original_delivery_note_id": ... }`

---

### 6. Event: `order_returned`

- Look up the original delivery note by `original_delivery_note_id` or by `order_id`
- If original delivery not found → return `404` with `delivery_not_found` error
- Resolve partner and products (same rules as shipment)
- Validate that return quantities do not exceed original delivery quantities → `422` if exceeded
- Create a receipt/return note that offsets the original delivery (may be partial)
- The receipt/return note must reference the original delivery
- Return `{ "receipt_note_id": ..., "original_delivery_note_id": ... }`

---

### 7. Product resolution

For each line:

1. If productCode provided → exact match
2. Else → match by SKU
3. If 0 matches → ERROR PRODUCT_NOT_FOUND
4. If >1 match → ERROR PRODUCT_AMBIGUOUS

NO fuzzy logic.

---

### 8. Partner resolution

Follow AUTO mode:

- direct channels → create/find customer
- platform channels → map to fixed partner

If mapping missing → ERROR PARTNER_MAPPING_MISSING

---

### 9. Delivery / receipt note creation

Reuse existing logic from:

functions/api/sales/deliveries.ts

For receipt/reversal and receipt/return notes, use the existing receipt note creation logic. If needed:
- extract shared helper (recommended)

---

### 10. Response format

Success (order_shipped):
```json
{
  "ok": true,
  "status": "created",
  "created": true,
  "replayed": false,
  "deliveryNo": "...",
  "deliveryId": "..."
}
```

Success (order_canceled_after_shipment / order_returned):
```json
{
  "ok": true,
  "status": "created",
  "created": true,
  "replayed": false,
  "receiptNoteId": "...",
  "receiptNoteNo": "...",
  "originalDeliveryNoteId": "..."
}
```

---

### 11. Integration audit UI

Build a read-only page for operators to monitor received integration events.

Data source: `integration_shipments` table.

Requirements:

- List view with filters: `event_type`, `status` (`created`, `replayed`, `failed`), `external_order_id`, date range.
- Detail view per event: payload, idempotency key, status, error info (if failed), linked document (delivery note or receipt note), original delivery reference (for cancel/return events).
- Click-through to the linked document in Accounting's existing UI (delivery note, receipt/reversal note, or receipt/return note).
- Summary counts: created, replayed, failed — total and per event type.
- **Read-only.** No edit, delete, or retry actions. This page is an audit/monitoring tool, not a control plane.

---

## CANCELLATION & RETURN POLICY — MUST FOLLOW

1. **Posted documents are immutable.** This integration never modifies or deletes a delivery note, receipt/reversal note, or receipt/return note once created.
2. **Cancellations after shipment produce compensating documents.** When `order_canceled_after_shipment` is received, create a receipt/reversal note that offsets the original delivery. Do not touch the original delivery.
3. **Returns produce compensating documents.** When `order_returned` is received, create a receipt/return note that offsets the original delivery. Do not touch the original delivery.
4. **Every compensating document must reference the original delivery** it offsets.
5. If a duplicate request arrives (same idempotency key, same payload), replay the original response — do not create, modify, or delete anything.
6. **No silent deletions.** Never delete or reverse a posted document without creating an explicit compensating document.
