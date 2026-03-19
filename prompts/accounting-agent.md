# Accounting Agent — Shipment Integration

You are working inside the **Accounting** repository only. Do not touch any other repository.

## Your task

Implement the server-side integration endpoint that receives shipment events from Sales Analyzer and creates delivery notes.

## What to implement

### 1. Database migration

Create a migration file that adds the `integration_idempotency_keys` table:

```sql
CREATE TABLE integration_idempotency_keys (
  key           VARCHAR(255) PRIMARY KEY,
  response_code INTEGER      NOT NULL,
  response_body JSONB        NOT NULL,
  created_at    TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

Provide the exact file path for the migration following this project's existing migration naming convention.

### 2. Route

Add a route at:

```
POST /api/integrations/sales-analyzer/shipments
```

Provide the exact file path where this route should be registered.

### 3. Controller / handler logic

Implement the following steps in order:

1. **Authenticate**: verify the `Authorization: Bearer <token>` header against the env var `INTEGRATION_TOKEN`. Return `401` if missing or invalid.
2. **Validate**: check that `Idempotency-Key` header is present and that the request body contains `order_id`, `shipped_at`, `customer_ref`, and a non-empty `lines` array. Return `400` if invalid.
3. **Idempotency check**: query `integration_idempotency_keys` by the key. If found, return the stored `response_code` and `response_body` immediately.
4. **Partner resolution**: look up the partner using `customer_ref` as an exact match. Return `404` with `{ "error": "partner_not_found", "customer_ref": "..." }` if not found.
5. **Product resolution**: for each line, look up the product using `product_ref` as an exact match. Return `404` with `{ "error": "product_not_found", "product_ref": "..." }` if not found. Validate the `unit` field; return `422` with `{ "error": "invalid_unit", ... }` if unrecognised.
6. **Create delivery note**: use the existing delivery note creation logic in this codebase. Do not duplicate or rewrite existing logic.
7. **Store idempotency key**: within the same database transaction as the delivery note creation, insert into `integration_idempotency_keys` with `response_code = 201` and the response body.
8. **Return `201`**: `{ "delivery_note_id": <id> }`.

### 4. Environment variable

Add `INTEGRATION_TOKEN` to the list of required environment variables (document in `.env.example` or equivalent).

### 5. Tests

Write tests covering all cases in `docs/08-test-plan.md` (Accounting section). Follow the testing conventions already used in this project.

## Assumptions to document

For each assumption you make about existing model names, column names, service methods, or file paths, write a short comment in the code or in a `## Assumptions` section in your reply.

## Output format

For every change, provide:
- Exact file path (relative to the repo root)
- Full file content or exact diff
- Migration SQL
- Any new env vars

Do not create Docker files, CI config, or package.json changes.
