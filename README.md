# shipment-sync-integration

Coordination and implementation workspace for the integration between **Sales Analyzer** and **Accounting**.

**Business goal:** When an order in Sales Analyzer becomes `shipped`, a delivery note is automatically created in Accounting.

This repo contains no runtime code — only planning documents and AI agent prompts.

---

## How to use this repo

Follow these steps in order:

### 1. Finalize the docs

Review and adjust the files in `docs/` to match your actual codebase conventions:

| File | Contents |
|------|----------|
| `docs/01-goal.md` | Business objective and success criteria |
| `docs/02-architecture.md` | System design and component responsibilities |
| `docs/03-api-contract.md` | Endpoint, headers, request/response shapes |
| `docs/04-partner-rules.md` | How Accounting resolves the customer/partner |
| `docs/05-product-rules.md` | How Accounting resolves products and units |
| `docs/06-idempotency-retries.md` | Idempotency key storage and retry schedule |
| `docs/07-rollout-plan.md` | Phased rollout and rollback procedure |
| `docs/08-test-plan.md` | Unit, integration, and end-to-end test cases |
| `docs/09-progress.md` | Implementation checklist — update as work progresses |

### 2. Run the Accounting agent first

Open `prompts/accounting-agent.md` in VS Code with your Accounting repo loaded.
Copy the prompt into your AI agent (e.g. GitHub Copilot Chat or similar).
The agent will implement:
- The idempotency table migration
- The `POST /api/integrations/sales-analyzer/shipments` endpoint
- Partner and product resolution
- Delivery note creation
- Auth middleware and tests

Review all generated code and migrations before applying them.

### 3. Review the Accounting output

- Confirm the migration SQL matches your ORM/migration conventions.
- Confirm the endpoint path, auth mechanism, and response codes match `docs/03-api-contract.md`.
- Run the Accounting tests.
- Note any assumptions the agent documented and verify them against your codebase.

### 4. Run the Sales Analyzer agent

Open `prompts/sales-analyzer-agent.md` in VS Code with your Sales Analyzer repo loaded.
Copy the prompt into your AI agent.
The agent will implement:
- The outbox table migration
- The hook that writes to the outbox when an order is shipped
- The background worker with retry logic
- Environment variable configuration and tests

Review all generated code before applying.

### 5. Test end-to-end

Follow the scenarios in `docs/08-test-plan.md` (End-to-End section) against a staging environment.
Once all scenarios pass, follow the rollout phases in `docs/07-rollout-plan.md`.
Track completion in `docs/09-progress.md`.
