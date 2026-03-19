# Product Resolution Rules

## Responsibility

Accounting is responsible for resolving each line item's product from the `product_ref` field.

## Resolution Logic

1. Look up the product in Accounting's product catalogue using `product_ref` as an **exact match** against the product's external reference field (e.g. `sku` or equivalent column).
2. If the product is found, use it.
3. If no product is found, return `404` with a clear error message:
   ```json
   { "error": "product_not_found", "product_ref": "SKU-1001" }
   ```

## No Fuzzy Matching

- Do **not** attempt name-based or partial matching.
- Do **not** auto-create products.
- The `product_ref` value must exist in Accounting's catalogue before the integration can process that line.

## Unit Validation

- The `unit` field must match a value in Accounting's unit-of-measure catalogue.
- If the unit is unrecognised, return `422`:
  ```json
  { "error": "invalid_unit", "unit": "box", "product_ref": "SKU-1001" }
  ```

## Applicability

Product resolution applies to all event types:

- `order_shipped`: resolve products to create delivery note lines.
- `order_canceled_after_shipment` and `order_returned`: resolve products to create receipt note lines. The same resolution rules apply.

## Partial Failures

- If **any** line fails resolution, the entire request is rejected. No document is created.
- Sales Analyzer treats `404` as a **permanent failure** (no retry).

## Setup Requirement

Before go-live, every product SKU used in Sales Analyzer must exist in Accounting with a matching external reference.
