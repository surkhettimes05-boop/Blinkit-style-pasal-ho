# Blinkit-style Pasalho Commerce Implementation

This repository is the implementation source-of-truth for the Pasalho customer web + mobile commerce rebuild.

## Target repositories

- **Core backend:** `surkhettimes05-boop/pasalho` — NestJS + Prisma + PostgreSQL inventory/order truth.
- **Customer web:** `surkhettimes05-boop/modern-trade-website` — Next.js 16 App Router.
- **Customer app:** `surkhettimes05-boop/modern-trade-app` — Flutter.
- **This repository:** architecture, technical specification, migration plan, API contracts, and Codex execution instructions.

## Primary document

Read **[TECHNICAL_IMPLEMENTATION_SPEC.md](./TECHNICAL_IMPLEMENTATION_SPEC.md)** before making code changes.

The target architecture is:

```
Customer Web ─┐
              ├── Commerce API (Pasalho NestJS)
Customer App ─┘
                     │
             Catalog / Pricing
             Serviceability
             Cart / Checkout
             SalesOrder
             StockReservation
             Fulfillment
             Payment
                     │
              Inventory Ledger
                     │
        Store-scoped InventorySnapshot
```

## Non-negotiable rules

1. Never use legacy `Product.stock` as customer-facing stock truth.
2. Never calculate authoritative prices or discounts in web/mobile clients.
3. Every storefront order must resolve a fulfillment store/location before reservation.
4. Every checkout write must be idempotent.
5. Every order reserves store inventory atomically before success is returned.
6. Web and Flutter use the same backend contracts.
7. Warehouse inventory is not customer-sellable inventory unless explicitly configured as a customer fulfillment location.
8. Do not create a parallel ecommerce inventory ledger or parallel order database.

## Execution order

Implementation is divided into **P0 launch-critical** and **P1 post-launch** work in the technical specification.

