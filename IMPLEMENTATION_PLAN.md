# Pasalho Blinkit-Style Commerce — Implementation Plan

**Date:** 2026-09-30  
**Source:** TECHNICAL_IMPLEMENTATION_SPEC.md  
**Execution branch:** `surkhettimes05-boop/pasalho:feat/blinkit-commerce-p0-foundation`

## Goal

Build the Pasalho customer-commerce foundation in dependency order so the Blinkit-style web and app sit on real store-scoped inventory, not legacy `Product.stock`.

## Delivery strategy

### Phase A — P0 foundation (current build)
1. **P0-01 Guardrails**
   - isolate customer commerce under `/api/v1/commerce`;
   - add commerce-specific errors/config;
   - deprecate legacy storefront checkout path;
   - keep employee JWT/RBAC untouched;
   - never introduce new storefront reads of `Product.stock`.

2. **P0-02 Customer identity**
   - customer phone/OTP identity;
   - separate customer JWT/session boundary;
   - rotating refresh tokens;
   - customer profile;
   - owned delivery-address CRUD;
   - additive Prisma migration.

3. **P0-03 Serviceability + fulfillment routing**
   - service zones;
   - zone-to-`InventoryLocation` mappings;
   - circle and polygon serviceability;
   - only active STORE locations eligible for P0;
   - deterministic provisional store routing;
   - public `POST /commerce/serviceability/resolve`.

### Phase B — inventory + checkout
4. P0-04 store-scoped catalog/pricing.
5. P0-05 FEFO reservations and concurrency hardening.
6. P0-06 server-authoritative cart.
7. P0-07 atomic idempotent checkout.

### Phase C — fulfillment
8. P0-08 order state machine/cancellation.
9. P0-09 picking/packing.
10. P0-10 COD, dispatch, reservation consumption, delivery close.

### Phase D — clients
11. P0-11 Next.js 16 customer web.
12. P0-12 Flutter customer app.

### Phase E — release
13. P0-13 full migration/build/concurrency/E2E gate and controlled beta.

## P0 foundation exit criteria

Phase A is accepted only when:
- customer auth cannot authenticate against employee/admin controllers;
- OTP challenge is rate-limited and one-time;
- refresh tokens rotate and are stored only as hashes;
- customers can only read/write their own addresses;
- serviceability rejects unsupported coordinates;
- only explicitly mapped, active STORE inventory locations are returned;
- routing is deterministic;
- Prisma migration is forward-only;
- existing employee auth remains unchanged;
- backend CI/build/test is green.

## Safety rules

- Do not modify historical migrations.
- Do not drop `Product.stock` yet.
- Do not build storefront UI before serviceability and store inventory contracts exist.
- Do not merge Phase A if Prisma validation/build/tests fail.
- Keep each implementation phase reviewable and reversible.
