# API Contract

## Endpoint

```
POST /api/integrations/sales-analyzer/events
```

This single endpoint handles all event types. The `event_type` field in the request body determines the action.

## Headers

| Header            | Required | Value                                                      |
|-------------------|----------|------------------------------------------------------------|
| `Content-Type`    | Yes      | `application/json`                                         |
| `Idempotency-Key` | Yes      | See format per event type below                            |
| `Authorization`   | Yes      | Bearer token shared between the two apps (env var)         |

## Supported Event Types

| `event_type` | Accounting action | Idempotency-Key format |
|--------------|-------------------|------------------------|
| `order_shipped` | Create delivery note | `sales-analyzer:order-shipped:<order_id>` |
| `order_canceled_after_shipment` | Create receipt/reversal note | `sales-analyzer:order-canceled:<order_id>` |
| `order_returned` | Create receipt/return note | `sales-analyzer:order-returned:<order_id>` |

---

## Event: `order_shipped`

### Request Body

```json
{
  "event_type": "order_shipped",
  "order_id": 42,
  "shipped_at": "2025-06-01T14:30:00Z",
  "customer_ref": "CUST-007",
  "lines": [
    {
      "product_ref": "SKU-1001",
      "quantity": 3,
      "unit": "pcs"
    }
  ]
}
```

### Field Descriptions

| Field               | Type            | Description                                              |
|---------------------|-----------------|----------------------------------------------------------|
| `event_type`        | string          | Must be `order_shipped`                                  |
| `order_id`          | integer         | Unique order ID from Sales Analyzer                      |
| `shipped_at`        | ISO 8601 string | UTC timestamp when the order was marked shipped          |
| `customer_ref`      | string          | Customer identifier used to resolve the partner          |
| `lines`             | array           | One entry per order line                                 |
| `lines[].product_ref` | string        | Product identifier used to resolve the product           |
| `lines[].quantity`  | number          | Shipped quantity                                         |
| `lines[].unit`      | string          | Unit of measure (must match Accounting's catalogue)      |

### Responses

| Status | Meaning |
|--------|---------|
| `201`  | Delivery note created. Body: `{ "delivery_note_id": 123 }` |
| `200`  | Duplicate request (idempotency hit). Body: `{ "delivery_note_id": 123 }` |

---

## Event: `order_canceled_after_shipment`

### Request Body

```json
{
  "event_type": "order_canceled_after_shipment",
  "order_id": 42,
  "canceled_at": "2025-06-05T10:00:00Z",
  "reason": "customer_request",
  "original_delivery_note_id": 123,
  "customer_ref": "CUST-007",
  "lines": [
    {
      "product_ref": "SKU-1001",
      "quantity": 3,
      "unit": "pcs"
    }
  ]
}
```

### Field Descriptions

| Field                        | Type            | Description                                              |
|------------------------------|-----------------|----------------------------------------------------------|
| `event_type`                 | string          | Must be `order_canceled_after_shipment`                  |
| `order_id`                   | integer         | Unique order ID from Sales Analyzer                      |
| `canceled_at`                | ISO 8601 string | UTC timestamp when the order was canceled                |
| `reason`                     | string          | Cancellation reason (informational)                      |
| `original_delivery_note_id`  | integer or null | Delivery note ID returned by the original shipment sync. If null, Accounting looks it up by `order_id`. |
| `customer_ref`               | string          | Customer identifier (same as original shipment)          |
| `lines`                      | array           | Lines to reverse (typically same as original delivery)   |

### Responses

| Status | Meaning |
|--------|---------|
| `201`  | Receipt/reversal note created. Body: `{ "receipt_note_id": 456, "original_delivery_note_id": 123 }` |
| `200`  | Duplicate request (idempotency hit). Same body as `201`. |
| `404`  | Original delivery note not found for this `order_id`. Do not retry — requires investigation. |

---

## Event: `order_returned`

### Request Body

```json
{
  "event_type": "order_returned",
  "order_id": 42,
  "returned_at": "2025-06-10T09:00:00Z",
  "reason": "defective",
  "original_delivery_note_id": 123,
  "customer_ref": "CUST-007",
  "lines": [
    {
      "product_ref": "SKU-1001",
      "quantity": 2,
      "unit": "pcs"
    }
  ]
}
```

### Field Descriptions

| Field                        | Type            | Description                                              |
|------------------------------|-----------------|----------------------------------------------------------|
| `event_type`                 | string          | Must be `order_returned`                                 |
| `order_id`                   | integer         | Unique order ID from Sales Analyzer                      |
| `returned_at`                | ISO 8601 string | UTC timestamp when the return was processed              |
| `reason`                     | string          | Return reason (informational)                            |
| `original_delivery_note_id`  | integer or null | Delivery note ID returned by the original shipment sync. If null, Accounting looks it up by `order_id`. |
| `customer_ref`               | string          | Customer identifier (same as original shipment)          |
| `lines`                      | array           | Lines being returned (quantity may be partial)           |

### Responses

| Status | Meaning |
|--------|---------|
| `201`  | Receipt/return note created. Body: `{ "receipt_note_id": 789, "original_delivery_note_id": 123 }` |
| `200`  | Duplicate request (idempotency hit). Same body as `201`. |
| `404`  | Original delivery note not found for this `order_id`. Do not retry — requires investigation. |

---

## Common Error Responses (all event types)

| Status | Meaning |
|--------|---------|
| `400`  | Invalid payload (missing fields, wrong types, unknown `event_type`). |
| `404`  | Partner, product, or original delivery note could not be resolved. |
| `409`  | Idempotency key reuse with a **different** payload. Indicates a bug — do not retry. |
| `422`  | Business rule violation (e.g. zero quantity, return quantity exceeds original). |
| `500`  | Unexpected server error — caller should retry with backoff. |

## Notes

- `4xx` errors (except `409` and `429`) are **not retried** by Sales Analyzer; they require investigation.
- `5xx` and network errors **are retried** with exponential backoff.
- The response body for `200` and `201` must be identical in structure for a given event.
- Compensating documents (receipt/reversal, receipt/return) must reference the `original_delivery_note_id` they offset.
- Posted documents are **never modified or deleted** by this integration. Corrections are always expressed as new compensating documents.
