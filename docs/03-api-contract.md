# API Contract

## Endpoint

```
POST /api/integrations/sales-analyzer/shipments
```

## Headers

| Header            | Required | Value                                                      |
|-------------------|----------|------------------------------------------------------------|
| `Content-Type`    | Yes      | `application/json`                                         |
| `Idempotency-Key` | Yes      | `sales-analyzer:order-shipped:<order_id>` (see below)      |
| `Authorization`   | Yes      | Bearer token shared between the two apps (env var)         |

### Idempotency-Key format

```
sales-analyzer:order-shipped:<order_id>
```

Example: `sales-analyzer:order-shipped:42`

## Request Body

```json
{
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
| `order_id`          | integer         | Unique order ID from Sales Analyzer                      |
| `shipped_at`        | ISO 8601 string | UTC timestamp when the order was marked shipped          |
| `customer_ref`      | string          | Customer identifier used to resolve the partner          |
| `lines`             | array           | One entry per order line                                 |
| `lines[].product_ref` | string        | Product identifier used to resolve the product           |
| `lines[].quantity`  | number          | Shipped quantity                                         |
| `lines[].unit`      | string          | Unit of measure (must match Accounting's catalogue)      |

## Responses

| Status | Meaning                                                                 |
|--------|-------------------------------------------------------------------------|
| `201`  | Delivery note created successfully. Body: `{ "delivery_note_id": 123 }` |
| `200`  | Duplicate request (idempotency hit). Body: `{ "delivery_note_id": 123 }` |
| `400`  | Invalid payload (missing fields, wrong types).                          |
| `404`  | Partner or product could not be resolved.                               |
| `409`  | Idempotency key reuse with a **different** payload. Should not occur in normal flow; indicates a bug in Sales Analyzer. Do not retry — requires investigation. |
| `422`  | Business rule violation (e.g. zero quantity).                           |
| `500`  | Unexpected server error — caller should retry with backoff.             |

## Notes

- `4xx` errors (except `409`) are **not retried** by Sales Analyzer; they require investigation.
- `5xx` and network errors **are retried** with exponential backoff.
- The response body for `200` and `201` must be identical in structure.
