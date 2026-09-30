# Codex Master Build Prompt — Pasalho Blinkit-Style Commerce

You are implementing Pasalho's customer web + mobile commerce system.

## Source of truth

Read this file first and follow it as the implementation contract:

`TECHNICAL_IMPLEMENTATION_SPEC.md`

Specification repository:
`surkhettimes05-boop/Blinkit-style-pasal-ho`

Implementation repositories:
- Backend: `surkhettimes05-boop/pasalho`
- Web: `surkhettimes05-boop/modern-trade-website`
- Flutter: `surkhettimes05-boop/modern-trade-app`

## Before changing code

1. Inspect the current default branch and working state of the target repository.
2. Read relevant existing modules, schema, migrations, tests, and repository instructions such as AGENTS.md.
3. Preserve unrelated work. Do not reset, discard, or rewrite unrelated changes.
4. Reconcile the implementation specification with live code when names differ, while preserving the architectural invariants below.
5. Do not silently invent a parallel architecture.

## Architectural invariants

- Never use legacy `Product.stock` as storefront stock truth.
- Customer availability comes from store-scoped `InventorySnapshot`.
- Reservations move AVAILABLE -> RESERVED atomically under PostgreSQL row locking.
- Keep the existing inventory ledger. Do not create a second inventory ledger.
- Extend existing `SalesOrder` with `source=STOREFRONT`; do not create a second ecommerce order engine.
- Fulfillment location must be resolved and validated before an order succeeds.
- Server owns pricing, discount, tax, totals, fulfillment, payment verification, and status transitions.
- Every checkout/cancellation/payment write is idempotent.
- Web and Flutter use the same `/api/v1/commerce` contracts.
- Do not expose warehouse inventory as customer-sellable unless explicitly configured as a fulfillment location.
- Do not split one P0 customer order across stores.
- Do not update storefront order status outside `OrderStateMachineService`.
- Do not hard-code secrets, OTPs, provider credentials, or production IDs in business logic.

## Execution batches

Do not implement all P0 work in one giant change.

### Batch A — Backend foundation

Implement **P0-01, P0-02, P0-03** only:
- storefront guardrails;
- customer identity/auth + addresses;
- serviceability + fulfillment routing.

Use additive Prisma migrations. Add tests. Run:
- Prisma validation;
- relevant PostgreSQL-backed tests;
- backend build/lint if configured.

Stop after P0-03 and report:
- exact files changed;
- migration names;
- test commands and results;
- commit hashes;
- unresolved blockers/risks.

### Batch B — Storefront inventory and checkout

Only after Batch A is accepted, implement:
- P0-04 catalog/pricing/availability;
- P0-05 storefront reservation hardening;
- P0-06 server cart;
- P0-07 atomic checkout.

This batch must include the concurrent oversell test specified in the implementation document.

### Batch C — Fulfillment

Only after Batch B is accepted:
- P0-08 order state machine/cancellation;
- P0-09 picking/packing;
- P0-10 COD/dispatch/inventory consumption.

### Batch D — Next.js web

Only after backend contracts are stable:
- P0-11 in `modern-trade-website`.

Do not create local pricing or inventory truth in Next.js.

### Batch E — Flutter

Only after backend contracts are stable:
- P0-12 in `modern-trade-app`.

Refactor incrementally toward Riverpod/Dio/go_router while keeping the app buildable.

### Batch F — Release gate

Run P0-13 end-to-end validation.

## High-risk current code to fix

The existing public storefront path in `pasalho` currently checks legacy `Product.stock`, stores customer details in order notes, uses a five-minute duplicate heuristic, and does not create a store-scoped inventory reservation.

That behavior must not remain in the production customer checkout path.

Migrate clients to the new commerce API and then deprecate/remove the legacy public-order path.

## Database discipline

- Never edit an already-applied historical migration.
- Use new forward-only migrations.
- Audit data before changing uniqueness constraints.
- For `InventorySnapshot`, verify data before adding `unitId` to the unique key.
- Test migrations against a representative database copy as well as a fresh database.
- Do not drop `Product.stock` in P0; remove storefront use first.

## Test discipline

The following tests are release-critical:
- concurrent checkout cannot oversell;
- failed reservation rolls the complete checkout transaction back;
- duplicate Idempotency-Key returns one logical order;
- same key + different request conflicts;
- cancellation restores sellable inventory;
- dispatch consumes a reservation once;
- customer cannot read another customer's order/address;
- employee fulfillment actions are branch/location scoped;
- COD becomes paid only after confirmed delivery.

Use PostgreSQL for row-lock/concurrency tests.

## Commit discipline

Use focused commits that map to the P0 item being implemented.

Do not combine unrelated formatting or refactors.

At the end of each execution batch, provide a compact implementation report and stop for review.
