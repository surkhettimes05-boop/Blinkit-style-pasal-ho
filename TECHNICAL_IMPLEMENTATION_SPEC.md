# Pasalho Blinkit-Style Web + App
## Technical Implementation Specification

**Status:** Build specification  
**Date:** 2026-09-30  
**Primary implementation repositories:**
- Backend: surkhettimes05-boop/pasalho
- Web: surkhettimes05-boop/modern-trade-website
- Flutter app: surkhettimes05-boop/modern-trade-app
- Architecture/spec source of truth: surkhettimes05-boop/Blinkit-style-pasal-ho

---

# 0. Purpose

This document converts the Pasalho customer-commerce PRD into an implementation contract that Codex can execute.

The target product is a Blinkit-style customer shopping experience adapted to Pasalho's operating model:

**location -> fulfillment store -> sellable store inventory -> catalog/search -> add -> cart -> checkout -> reservation -> picking -> packing -> dispatch -> delivery**

This is not a separate ecommerce database. The customer web and Flutter app are client surfaces on top of Pasalho OS.

---

# 1. Current-state audit that this implementation must respect

The current Pasalho backend already contains useful foundations:

- NestJS app with global prefix /api/v1.
- Prisma/PostgreSQL.
- Branch and InventoryLocation scoping.
- InventorySnapshot with stock states including AVAILABLE and RESERVED.
- InventoryEvent and InventoryMovement immutable ledger concepts.
- StockReservation and StockReservationItem models.
- StockReservationService that moves quantity from AVAILABLE to RESERVED under row locks.
- SalesOrder / SalesOrderItem.
- OrderSource includes STOREFRONT.
- IdempotencyRecord exists.
- Delivery models exist.
- Catalog module exists.
- ScopeGuard and permission guards exist.
- Current web is Next.js 16 App Router.
- Current mobile app is Flutter.

The current public storefront order path is not production-safe for the new product. It currently:

- checks legacy Product.stock;
- does not resolve a fulfillment InventoryLocation;
- does not reserve inventory;
- uses a five-minute duplicate heuristic instead of a real Idempotency-Key;
- serializes customer data into SalesOrder.notes;
- has an incomplete customer order state machine.

Therefore the first implementation objective is not UI. It is to make STOREFRONT orders obey the same inventory truth as Pasalho OS.

---

# 2. Non-negotiable architecture rules

1. Product.stock is deprecated for storefront decisions. No new commerce code may read it.
2. InventorySnapshot is the read model for inventory availability.
3. InventoryEvent / InventoryMovement remain the inventory audit trail.
4. Storefront stock is always scoped by fulfillment InventoryLocation.
5. An order is successful only after the required inventory has been reserved atomically.
6. Web and Flutter use the same HTTP API contracts.
7. Client-provided price, discount, tax, total, fulfillment store, payment success, or stock values are never authoritative.
8. Checkout writes are idempotent.
9. Order status transitions are server-controlled.
10. Warehouse stock is not customer-sellable unless the location is explicitly enabled for customer fulfillment.
11. Do not create a second order engine. Extend SalesOrder for STOREFRONT.
12. Do not create a second inventory ledger.
13. Every storefront order must have a customer or guest identity, an immutable delivery-address snapshot, a fulfillment location, and an order-status history.
14. The customer-facing API must never expose internal RBAC or warehouse implementation details unnecessarily.

---

# 3. Target system topology

~~~text
                         PASALHO CUSTOMER

                +---------------------------+
                |                           |
          Next.js Web                 Flutter App
                |                           |
                +-------------+-------------+
                              |
                    /api/v1/commerce
                              |
        +---------------------+----------------------+
        |                     |                      |
 Customer/Auth        Catalog/Search          Cart/Checkout
        |                     |                      |
        +---------------------+----------------------+
                              |
                       SalesOrder
                              |
                   Fulfillment Routing
                              |
                     StockReservation
                              |
              InventorySnapshot + Ledger
                              |
             Store InventoryLocation truth
                              |
                 Picking / Packing / Delivery
~~~

---

# 4. Repository responsibilities

## 4.1 pasalho backend

Target path: backend/src/commerce

The existing admin/catalog/inventory/sales modules remain. Add a customer-commerce boundary so public APIs are not mixed with internal admin APIs.

Required module tree:

~~~text
backend/src/commerce/
  commerce.module.ts

  customer-auth/
    customer-auth.module.ts
    customer-auth.controller.ts
    customer-auth.service.ts
    customer-jwt.guard.ts
    optional-customer-jwt.guard.ts
    customer-token.service.ts
    dto/
      request-otp.dto.ts
      verify-otp.dto.ts
      refresh-customer-token.dto.ts

  customers/
    customers.module.ts
    customers.controller.ts
    customers.service.ts
    dto/
      update-customer.dto.ts
      create-address.dto.ts
      update-address.dto.ts

  serviceability/
    serviceability.module.ts
    serviceability.controller.ts
    serviceability.service.ts
    fulfillment-router.service.ts
    geo.service.ts
    dto/
      resolve-serviceability.dto.ts

  storefront-catalog/
    storefront-catalog.module.ts
    storefront-catalog.controller.ts
    storefront-catalog.service.ts
    availability.service.ts
    pricing.service.ts
    dto/
      catalog-query.dto.ts
      search-query.dto.ts

  carts/
    carts.module.ts
    carts.controller.ts
    carts.service.ts
    cart-pricing.service.ts
    dto/
      create-cart.dto.ts
      add-cart-item.dto.ts
      update-cart-item.dto.ts
      revalidate-cart.dto.ts

  checkout/
    checkout.module.ts
    checkout.controller.ts
    checkout.service.ts
    storefront-order.service.ts
    storefront-reservation.service.ts
    order-state-machine.service.ts
    dto/
      checkout-preview.dto.ts
      place-order.dto.ts
      cancel-order.dto.ts

  fulfillment/
    fulfillment.module.ts
    fulfillment.controller.ts
    fulfillment.service.ts
    picking.service.ts
    dto/
      start-picking.dto.ts
      pick-item.dto.ts
      report-shortage.dto.ts
      transition-order.dto.ts

  commerce-payments/
    commerce-payments.module.ts
    commerce-payments.controller.ts
    commerce-payments.service.ts
    dto/
      initiate-payment.dto.ts
      confirm-cod.dto.ts

  commerce-analytics/
    commerce-analytics.module.ts
    commerce-analytics.controller.ts
    commerce-analytics.service.ts
    dto/
      track-event.dto.ts
~~~

P0 includes all folders except commerce-analytics may be minimal. P1 completes analytics, loyalty, promotion, push, recommendation, and advanced search.

## 4.2 modern-trade-website

Use the existing Next.js 16 App Router under frontend/src/app.

The web should become a thin customer client. No duplicated catalog database or pricing engine.

## 4.3 modern-trade-app

The current Flutter app already has home, catalog, product, cart, checkout, orders, account, addresses and loyalty surfaces. Refactor rather than rewrite blindly.

P0 architecture should migrate toward:

- flutter_riverpod
- dio
- go_router
- freezed/json_serializable if code generation is accepted
- drift only for local cache/offline-safe read data, not as order truth
- flutter_secure_storage for tokens

The current http + monolithic AppState approach should be replaced incrementally.

---

# 5. Prisma target model

## 5.1 Key modeling decision

Existing Product remains the SKU-level inventory object.

Do not rename Product to SKU in P0 because it would cause unnecessary migration risk across inventory, sales, transfers and POS.

Add ProductGroup to group consumer-visible variants.

Example:

Coca-Cola is ProductGroup.

Coca-Cola 250 ml, 500 ml, 1 L are individual Product rows with independent SKU codes, barcodes, inventory and prices.

---

# 6. Prisma enums to add or extend

Add:

~~~prisma
enum CustomerStatus {
  ACTIVE
  BLOCKED
  DELETED
}

enum CustomerAddressLabel {
  HOME
  WORK
  OTHER
}

enum ServiceZoneStatus {
  ACTIVE
  INACTIVE
}

enum CartStatus {
  ACTIVE
  CONVERTED
  ABANDONED
  EXPIRED
}

enum CommercePaymentMethod {
  COD
  ESEWA
  KHALTI
  BANK
}

enum CommercePaymentStatus {
  PENDING
  AUTHORIZED
  PAID
  FAILED
  CANCELLED
  REFUND_PENDING
  REFUNDED
  PARTIALLY_REFUNDED
}

enum OrderStatusEventActor {
  CUSTOMER
  STAFF
  SYSTEM
  RIDER
}

enum ProductRequestStatus {
  OPEN
  REVIEWED
  SOURCED
  REJECTED
}

enum FulfillmentShortageAction {
  REMOVE_ITEM
  CONTACT_CUSTOMER
  SUBSTITUTE
}

enum PromotionType {
  PERCENT_OFF
  FIXED_OFF
  BUY_X_GET_Y
  BUNDLE
  FREE_DELIVERY
}

enum LoyaltyEntryType {
  EARN
  REDEEM
  REVERSAL
  ADJUSTMENT
}
~~~

Extend SalesOrderStatus to:

~~~prisma
enum SalesOrderStatus {
  DRAFT
  CONFIRMED
  INVOICED

  PLACED
  PICKING
  PACKED
  OUT_FOR_DELIVERY
  DELIVERED

  PARTIALLY_FULFILLED
  CANCELLED
  FAILED
  REFUND_PENDING
  REFUNDED
}
~~~

Do not remove existing statuses because DNP/sales-rep flows already use them.

---

# 7. Prisma new models

Add the following models.

~~~prisma
model Customer {
  id          String         @id @default(uuid())
  phone       String         @unique
  fullName    String?
  email       String?        @unique
  status      CustomerStatus @default(ACTIVE)
  lastLoginAt DateTime?
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt
  deletedAt   DateTime?

  sessions    CustomerSession[]
  addresses   CustomerAddress[]
  carts       Cart[]
  orders      SalesOrder[]
  payments    CommercePayment[]
  productRequests ProductRequest[]
  loyaltyAccount LoyaltyAccount?
  devices     CustomerDevice[]

  @@index([status])
  @@index([createdAt])
}

model CustomerSession {
  id               String   @id @default(uuid())
  customerId       String
  refreshTokenHash String
  refreshExpiresAt DateTime
  deviceId         String?
  userAgent        String?
  ipAddress        String?
  createdAt        DateTime @default(now())
  revokedAt        DateTime?

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Cascade)

  @@index([customerId])
  @@index([refreshExpiresAt])
}

model CustomerAddress {
  id           String               @id @default(uuid())
  customerId   String
  label        CustomerAddressLabel @default(HOME)
  customLabel  String?
  recipientName String?
  phone        String?
  province     String?
  district     String?
  municipality String?
  ward         String?
  area         String
  street       String?
  landmark     String?
  latitude     Decimal?             @db.Decimal(10, 7)
  longitude    Decimal?             @db.Decimal(10, 7)
  instructions String?
  isDefault    Boolean              @default(false)
  createdAt    DateTime             @default(now())
  updatedAt    DateTime             @updatedAt
  deletedAt    DateTime?

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Cascade)
  orders   SalesOrder[]

  @@index([customerId])
  @@index([isDefault])
}

model ServiceZone {
  id            String            @id @default(uuid())
  code          String            @unique
  name          String
  status        ServiceZoneStatus @default(ACTIVE)
  polygon       Json?
  centerLat     Decimal?          @db.Decimal(10, 7)
  centerLng     Decimal?          @db.Decimal(10, 7)
  radiusMeters  Int?
  minOrder      Decimal           @default(0) @db.Decimal(18, 4)
  deliveryFee   Decimal           @default(0) @db.Decimal(18, 4)
  freeDeliveryThreshold Decimal?  @db.Decimal(18, 4)
  etaMinMinutes Int               @default(30)
  etaMaxMinutes Int               @default(60)
  createdAt     DateTime          @default(now())
  updatedAt     DateTime          @updatedAt

  locations ServiceZoneFulfillmentLocation[]

  @@index([status])
}

model ServiceZoneFulfillmentLocation {
  id                String @id @default(uuid())
  serviceZoneId     String
  inventoryLocationId String
  priority          Int    @default(100)
  isEnabled         Boolean @default(true)
  maxActiveOrders   Int?

  serviceZone       ServiceZone       @relation(fields: [serviceZoneId], references: [id], onDelete: Cascade)
  inventoryLocation InventoryLocation @relation(fields: [inventoryLocationId], references: [id], onDelete: Restrict)

  @@unique([serviceZoneId, inventoryLocationId])
  @@index([inventoryLocationId])
  @@index([serviceZoneId, isEnabled])
}

model ProductGroup {
  id          String   @id @default(uuid())
  slug        String   @unique
  name        String
  description String?
  categoryId  String
  brandId     String?
  imageUrl    String?
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  category Category @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  brand    Brand?   @relation(fields: [brandId], references: [id], onDelete: SetNull)
  products Product[]

  @@index([categoryId])
  @@index([brandId])
  @@index([isActive])
}

model ProductImage {
  id        String   @id @default(uuid())
  productId String
  url       String
  altText   String?
  sortOrder Int      @default(0)
  createdAt DateTime @default(now())

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@index([productId, sortOrder])
}

model StoreProductConfig {
  id                String   @id @default(uuid())
  inventoryLocationId String
  productId         String
  isVisible         Boolean  @default(true)
  sellingPrice      Decimal? @db.Decimal(18, 4)
  mrp               Decimal? @db.Decimal(18, 4)
  safetyStockBaseQty Decimal @default(0) @db.Decimal(18, 6)
  maxOrderBaseQty   Decimal? @db.Decimal(18, 6)
  sortRank          Int?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  inventoryLocation InventoryLocation @relation(fields: [inventoryLocationId], references: [id], onDelete: Cascade)
  product           Product           @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([inventoryLocationId, productId])
  @@index([productId])
  @@index([inventoryLocationId, isVisible])
}

model Cart {
  id                    String     @id @default(uuid())
  cartToken             String     @unique
  customerId            String?
  inventoryLocationId   String
  serviceZoneId         String?
  status                CartStatus @default(ACTIVE)
  currency              String     @default("NPR")
  subtotal              Decimal    @default(0) @db.Decimal(18, 4)
  discountTotal         Decimal    @default(0) @db.Decimal(18, 4)
  deliveryFee           Decimal    @default(0) @db.Decimal(18, 4)
  grandTotal            Decimal    @default(0) @db.Decimal(18, 4)
  version               Int        @default(1)
  expiresAt             DateTime
  convertedOrderId      String?    @unique
  createdAt             DateTime   @default(now())
  updatedAt             DateTime   @updatedAt

  customer              Customer?         @relation(fields: [customerId], references: [id], onDelete: SetNull)
  inventoryLocation     InventoryLocation @relation(fields: [inventoryLocationId], references: [id], onDelete: Restrict)
  serviceZone           ServiceZone?      @relation(fields: [serviceZoneId], references: [id], onDelete: SetNull)
  items                 CartItem[]
  convertedOrder        SalesOrder?       @relation("ConvertedCartOrder", fields: [convertedOrderId], references: [id], onDelete: SetNull)

  @@index([customerId, status])
  @@index([inventoryLocationId])
  @@index([expiresAt])
}

model CartItem {
  id        String   @id @default(uuid())
  cartId    String
  productId String
  unitId    String
  quantity  Decimal  @db.Decimal(18, 6)
  addedAt   DateTime @default(now())
  updatedAt DateTime @updatedAt

  cart    Cart    @relation(fields: [cartId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id], onDelete: Restrict)
  unit    Unit    @relation(fields: [unitId], references: [id], onDelete: Restrict)

  @@unique([cartId, productId, unitId])
  @@index([cartId])
}

model CommercePayment {
  id                String                @id @default(uuid())
  paymentNo         String                @unique
  customerId        String
  salesOrderId      String
  method            CommercePaymentMethod
  status            CommercePaymentStatus @default(PENDING)
  amount            Decimal               @db.Decimal(18, 4)
  provider          String?
  providerReference String?
  providerPayload   Json?
  idempotencyKey    String?               @unique
  authorizedAt      DateTime?
  paidAt            DateTime?
  failedAt          DateTime?
  refundedAt        DateTime?
  createdAt         DateTime              @default(now())
  updatedAt         DateTime              @updatedAt

  customer   Customer   @relation(fields: [customerId], references: [id], onDelete: Restrict)
  salesOrder SalesOrder @relation(fields: [salesOrderId], references: [id], onDelete: Restrict)

  @@index([customerId])
  @@index([salesOrderId])
  @@index([status])
}

model OrderStatusEvent {
  id           String                @id @default(uuid())
  salesOrderId String
  fromStatus   SalesOrderStatus?
  toStatus     SalesOrderStatus
  actorType    OrderStatusEventActor
  actorUserId  String?
  customerId   String?
  reasonCode   String?
  note         String?
  metadata     Json?
  createdAt    DateTime              @default(now())

  salesOrder SalesOrder @relation(fields: [salesOrderId], references: [id], onDelete: Cascade)
  actorUser  User?       @relation(fields: [actorUserId], references: [id], onDelete: SetNull)
  customer   Customer?   @relation(fields: [customerId], references: [id], onDelete: SetNull)

  @@index([salesOrderId, createdAt])
}

model ProductRequest {
  id         String               @id @default(uuid())
  customerId String?
  query      String
  status     ProductRequestStatus @default(OPEN)
  metadata   Json?
  createdAt  DateTime             @default(now())
  updatedAt  DateTime             @updatedAt

  customer Customer? @relation(fields: [customerId], references: [id], onDelete: SetNull)

  @@index([status])
  @@index([createdAt])
}

model CustomerDevice {
  id           String   @id @default(uuid())
  customerId   String
  deviceId     String
  platform     String
  pushToken    String?
  lastActiveAt DateTime @default(now())
  createdAt    DateTime @default(now())

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Cascade)

  @@unique([customerId, deviceId])
  @@index([pushToken])
}

model LoyaltyAccount {
  id           String   @id @default(uuid())
  customerId   String   @unique
  pointsBalance Int     @default(0)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Cascade)
  entries  LoyaltyEntry[]
}

model LoyaltyEntry {
  id              String           @id @default(uuid())
  loyaltyAccountId String
  type            LoyaltyEntryType
  points          Int
  salesOrderId    String?
  reason          String?
  createdAt       DateTime         @default(now())

  loyaltyAccount LoyaltyAccount @relation(fields: [loyaltyAccountId], references: [id], onDelete: Cascade)
  salesOrder     SalesOrder?    @relation(fields: [salesOrderId], references: [id], onDelete: SetNull)

  @@index([loyaltyAccountId, createdAt])
}

model Promotion {
  id          String        @id @default(uuid())
  code        String?       @unique
  name        String
  type        PromotionType
  value       Decimal?      @db.Decimal(18, 4)
  minSpend    Decimal?      @db.Decimal(18, 4)
  startsAt    DateTime
  endsAt      DateTime
  isActive    Boolean       @default(true)
  rules       Json?
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  @@index([isActive, startsAt, endsAt])
}
~~~

Promotion, LoyaltyAccount, LoyaltyEntry, CustomerDevice and ProductRequest are P1-capable models. They may be migrated after P0 if minimizing the first migration set is preferred.

---

# 8. Existing Prisma model modifications

## 8.1 Product

Add:

~~~prisma
productGroupId      String?
slug                String?  @unique
shortDescription    String?
searchKeywords      String[] @default([])
storefrontVisible   Boolean  @default(false)

productGroup        ProductGroup?       @relation(fields: [productGroupId], references: [id], onDelete: SetNull)
images              ProductImage[]
storeConfigs        StoreProductConfig[]
cartItems           CartItem[]

@@index([productGroupId])
@@index([storefrontVisible, isActive])
~~~

Do not remove current imageUrl in P0. Treat it as the legacy primary image fallback while ProductImage is populated.

## 8.2 Category

Add:

~~~prisma
slug          String? @unique
imageUrl      String?
sortRank      Int?
productGroups ProductGroup[]
~~~

## 8.3 Brand

Add:

~~~prisma
slug          String? @unique
logoUrl       String?
productGroups ProductGroup[]
~~~

## 8.4 InventoryLocation

Add relations:

~~~prisma
serviceZones       ServiceZoneFulfillmentLocation[]
storeProductConfigs StoreProductConfig[]
carts              Cart[]
storefrontOrders   SalesOrder[] @relation("StorefrontFulfillmentLocation")
~~~

No inventory quantity field is added here.

## 8.5 InventorySnapshot

Change the unique constraint from:

~~~prisma
@@unique([locationId, productId, batchId, stockState])
~~~

to:

~~~prisma
@@unique([locationId, productId, batchId, stockState, unitId])
~~~

Reason: StockReservationService reads and locks snapshots by unitId. The database uniqueness key must match the service's unit-scoped semantics.

Before applying the unique-index migration, run a duplicate/conflict audit and normalize any rows that would collide.

## 8.6 SalesOrder

Add:

~~~prisma
customerId             String?
customerAddressId      String?
fulfillmentLocationId  String?
cartId                  String? @unique
deliveryAddressSnapshot Json?
deliveryInstructions   String?
deliveryFee            Decimal @default(0) @db.Decimal(18, 4)
handlingFee            Decimal @default(0) @db.Decimal(18, 4)
couponDiscount         Decimal @default(0) @db.Decimal(18, 4)
placedAt               DateTime?
pickingStartedAt       DateTime?
packedAt               DateTime?
outForDeliveryAt       DateTime?
deliveredAt            DateTime?
cancelledAt            DateTime?
cancellationReason     String?

customer            Customer?         @relation(fields: [customerId], references: [id], onDelete: SetNull)
customerAddress     CustomerAddress?  @relation(fields: [customerAddressId], references: [id], onDelete: SetNull)
fulfillmentLocation InventoryLocation? @relation("StorefrontFulfillmentLocation", fields: [fulfillmentLocationId], references: [id], onDelete: Restrict)
convertedFromCart   Cart?              @relation("ConvertedCartOrder")
commercePayments    CommercePayment[]
statusEvents        OrderStatusEvent[]
loyaltyEntries      LoyaltyEntry[]

@@index([customerId, createdAt])
@@index([source, status, createdAt])
@@index([fulfillmentLocationId, status])
~~~

For a STOREFRONT order:
- branchId is the fulfillment branch.
- fulfillmentLocationId is mandatory.
- customerId is mandatory once checkout is submitted.
- deliveryAddressSnapshot is mandatory.
- createdById remains the system actor for compatibility with existing audit relations.

The system actor must come from configuration or seed lookup, never a hard-coded UUID inside business logic.

## 8.7 StockReservation

Add:

~~~prisma
expiresAt DateTime?

@@index([salesOrderId, status])
@@index([expiresAt, status])
~~~

P0 uses an order-level reservation TTL while an order is PLACED. Staff confirmation/picking extends or removes the TTL.

## 8.8 StockReservationItem

Remove:

~~~prisma
@@unique([reservationId, salesOrderItemId])
~~~

Replace with:

~~~prisma
@@index([reservationId, salesOrderItemId])
@@index([reservationId, productId, batchId, unitId])
~~~

Reason: one storefront order item may need allocation from multiple batches. The reservation allocation is allowed to split across batches using FEFO.

## 8.9 Invoice

To allow invoices sourced from a STORE InventoryLocation without manufacturing a fake warehouse:

Change:

~~~prisma
warehouseId String
warehouse   Warehouse
~~~

to:

~~~prisma
warehouseId String?
warehouse   Warehouse? @relation(fields: [warehouseId], references: [id], onDelete: Restrict)
~~~

Keep sourceLocationId mandatory.

This migration must be verified against all current InvoiceService assumptions before code rollout.

## 8.10 DeliveryItem

Change retailerId to nullable for direct-customer orders and create a proper relation to SalesOrder:

~~~prisma
retailerId String?
orderId    String?

retailer Retailer?   @relation(fields: [retailerId], references: [id], onDelete: Restrict)
order    SalesOrder? @relation(fields: [orderId], references: [id], onDelete: SetNull)

@@index([orderId])
~~~

Existing B2B delivery items remain valid.

---

# 9. Migration sequence

Do not generate one giant migration.

Use ordered, reviewable migrations.

## Migration 1
Name:

202609300001_storefront_identity_serviceability

Contains:
- Customer
- CustomerSession
- CustomerAddress
- ServiceZone
- ServiceZoneFulfillmentLocation
- customer-related enums
- relations only

Backfill:
- none required

Acceptance:
- prisma validate
- prisma migrate deploy against fresh DB
- migrate against copy of production/staging DB
- no existing row changes

## Migration 2
Name:

202609300002_storefront_catalog_listing

Contains:
- ProductGroup
- ProductImage
- StoreProductConfig
- Product fields
- Category slug/image/sort
- Brand slug/logo

Backfill:
- generate deterministic slugs
- set Product.storefrontVisible from current isActive AND existing storefront data where safe
- create StoreProductConfig only for explicit storefront fulfillment locations

Do not blindly make all warehouse products visible.

## Migration 3
Name:

202609300003_storefront_inventory_constraints

Contains:
- InventorySnapshot unique-key change including unitId
- StockReservation.expiresAt
- StockReservationItem index change

Before migration:
- run SQL report for duplicate snapshots under new key
- abort migration if conflicts cannot be reconciled deterministically

## Migration 4
Name:

202609300004_storefront_cart_order

Contains:
- Cart
- CartItem
- SalesOrder storefront fields
- SalesOrderStatus additions
- OrderStatusEvent

Backfill:
- existing STOREFRONT orders may remain customerId null
- parse old notes into deliveryAddressSnapshot only if safely parseable
- do not fabricate customer records from ambiguous notes automatically

## Migration 5
Name:

202609300005_storefront_payment_delivery

Contains:
- CommercePayment
- Invoice.warehouseId nullable
- DeliveryItem retailerId nullable
- DeliveryItem order relation

## Migration 6 - P1
Name:

202609300006_storefront_growth

Contains:
- Promotion
- ProductRequest
- CustomerDevice
- LoyaltyAccount
- LoyaltyEntry

## Product.stock deprecation

Do not drop Product.stock in P0.

Instead:
1. mark it deprecated in schema comment;
2. remove all STOREFRONT reads/writes;
3. add grep/static tests that fail when new storefront code uses product.stock;
4. remove the field only after POS/catalog legacy paths are migrated.

---

# 10. Storefront inventory semantics

Customer availability is:

~~~text
sellable base quantity
= SUM InventorySnapshot.baseQuantity
  where locationId = selected fulfillment location
  and productId = selected SKU
  and stockState = AVAILABLE
  and batch is usable/non-expired
- StoreProductConfig.safetyStockBaseQty
~~~

Reserved inventory is already moved from AVAILABLE to RESERVED by the inventory ledger workflow, so do not subtract reservations again if InventorySnapshot state transitions are functioning correctly.

Customer-facing states:

- AVAILABLE: sellableBaseQty > low-stock threshold
- LOW_STOCK: 0 < sellableBaseQty <= threshold
- OUT_OF_STOCK: sellableBaseQty <= 0

Exact inventory quantity should remain internal by default.

---

# 11. FEFO allocation for storefront reservations

The customer never chooses inventory batch.

For batch-tracked products:

1. query AVAILABLE snapshots at the fulfillment location;
2. join Batch;
3. reject BLOCKED or EXPIRED batch status;
4. reject expiryDate <= now;
5. order by expiryDate ASC NULLS LAST, then createdAt ASC;
6. allocate requested base quantity across one or more snapshots;
7. lock rows FOR UPDATE;
8. move each allocation AVAILABLE -> RESERVED;
9. create one StockReservationItem per allocation;
10. if complete quantity cannot be allocated, roll back the entire checkout transaction.

For non-batch-tracked products, allocate from batchId null.

Do not reserve partially and then return an error.

---

# 12. StorefrontReservationService contract

Add:

~~~ts
reserveOrder(
  tx: Prisma.TransactionClient,
  input: {
    salesOrderId: string;
    branchId: string;
    locationId: string;
    createdById: string;
    expiresAt: Date;
  }
): Promise<StockReservation>

releaseOrder(
  tx: Prisma.TransactionClient,
  input: {
    salesOrderId: string;
    actorUserId: string;
    reason: string;
  }
): Promise<void>

consumeOrder(
  tx: Prisma.TransactionClient,
  input: {
    salesOrderId: string;
    branchId: string;
    actorUserId: string;
    referenceType: "SALES_ORDER" | "INVOICE";
    referenceId: string;
  }
): Promise<void>
~~~

reserveOrder must create StockReservation and StockReservationItem rows as well as inventory state movements.

Do not rely only on RESERVED snapshots because order-to-reservation attribution is required for cancellation, auditing and consumption.

---

# 13. Fulfillment routing

## P0 rule

Input:
- customer lat/lng or address-associated coordinates
- active service zones
- active fulfillment locations in the matching zone

Candidate scoring:

~~~text
candidate is eligible only when:
- ServiceZone ACTIVE
- mapping isEnabled
- InventoryLocation ACTIVE
- InventoryLocation.type = STORE unless explicitly configured otherwise
- store capacity not exceeded, if maxActiveOrders exists

score:
1. stock completeness DESC
2. ServiceZoneFulfillmentLocation.priority ASC
3. geographic distance ASC
~~~

For P0, prefer a single store that can fulfill the complete cart.

Do not split one customer order across stores.

If no store can fulfill the complete cart:
- return CART_NOT_FULFILLABLE;
- include item-level unavailable products;
- do not silently route some items elsewhere.

P1 may introduce partial fulfillment or multi-store behavior only after economics justify it.

---

# 14. Geo behavior

P0 supports:
- circular service zone using centerLat, centerLng, radiusMeters;
- polygon service zone using GeoJSON-like polygon stored as JSON.

GeoService exposes:

~~~ts
containsPoint(zone: ServiceZone, lat: number, lng: number): boolean
distanceMeters(aLat: number, aLng: number, bLat: number, bLng: number): number
~~~

No PostGIS is required for Birendranagar pilot.

If zone count/traffic becomes large, migrate to PostGIS later.

---

# 15. Pricing truth

StorefrontPricingService determines a line price using this precedence:

1. active server-side Promotion result, P1;
2. StoreProductConfig.sellingPrice if set;
3. Product.sellingPrice;
4. Product.mrp only as fallback if business rules permit;
5. otherwise product is not orderable.

MRP precedence:

1. StoreProductConfig.mrp;
2. Product.mrp.

The server returns:

- unitPrice
- mrp
- discountAmount
- lineTotal

The client sends only:
- productId
- unitId
- quantity

Never accept lineTotal from client as authoritative.

---

# 16. Public API namespace

All customer-commerce APIs use:

/api/v1/commerce

Existing global ResponseInterceptor wraps normal responses:

~~~json
{
  "success": true,
  "data": {}
}
~~~

Errors continue through the existing HttpExceptionFilter/AppError system.

---

# 17. Authentication API

## POST /api/v1/commerce/auth/request-otp

Request:

~~~json
{
  "phone": "98XXXXXXXX"
}
~~~

Response:

~~~json
{
  "challengeId": "uuid",
  "expiresInSeconds": 300
}
~~~

P0 may use a development OTP provider behind an interface, but production must use a real SMS/OTP provider before public launch.

Rate limits:
- 3 requests/minute/phone
- 10 requests/hour/phone
- IP throttling

## POST /api/v1/commerce/auth/verify-otp

Request:

~~~json
{
  "challengeId": "uuid",
  "phone": "98XXXXXXXX",
  "otp": "123456",
  "deviceId": "optional-device-id"
}
~~~

Response:

~~~json
{
  "accessToken": "...",
  "refreshToken": "...",
  "expiresIn": 900,
  "customer": {
    "id": "uuid",
    "phone": "98XXXXXXXX",
    "fullName": null
  }
}
~~~

If phone does not exist, create Customer atomically after OTP verification.

## POST /api/v1/commerce/auth/refresh

Rotating refresh token.

## POST /api/v1/commerce/auth/logout

Revokes current CustomerSession.

---

# 18. Customer API

## GET /api/v1/commerce/me

Auth required.

## PATCH /api/v1/commerce/me

Allowed fields:
- fullName
- email

## GET /api/v1/commerce/me/addresses

## POST /api/v1/commerce/me/addresses

Request:

~~~json
{
  "label": "HOME",
  "recipientName": "Name",
  "phone": "98XXXXXXXX",
  "province": "Karnali",
  "district": "Surkhet",
  "municipality": "Birendranagar",
  "ward": "6",
  "area": "Example Area",
  "street": "Example Street",
  "landmark": "Near ...",
  "latitude": 28.60,
  "longitude": 81.63,
  "instructions": "Call before arrival",
  "isDefault": true
}
~~~

## PATCH /api/v1/commerce/me/addresses/:addressId

## DELETE /api/v1/commerce/me/addresses/:addressId

Soft delete.

---

# 19. Serviceability API

## POST /api/v1/commerce/serviceability/resolve

Can be called without login.

Request:

~~~json
{
  "latitude": 28.60,
  "longitude": 81.63
}
~~~

Response when serviceable:

~~~json
{
  "serviceable": true,
  "serviceZone": {
    "id": "zone-id",
    "name": "Birendranagar Core"
  },
  "fulfillment": {
    "locationId": "store-location-id",
    "branchId": "branch-id",
    "storeName": "Pasalho Birendranagar"
  },
  "delivery": {
    "etaMinMinutes": 30,
    "etaMaxMinutes": 50,
    "deliveryFee": 60,
    "freeDeliveryThreshold": 1500,
    "minOrder": 300
  }
}
~~~

Response when not serviceable:

~~~json
{
  "serviceable": false,
  "reason": "OUTSIDE_SERVICE_ZONE"
}
~~~

The returned location is a provisional shopping location. Checkout re-runs routing and stock validation.

---

# 20. Catalog API

All catalog endpoints require a fulfillment location context.

Use either:
- query param locationId, or
- X-Fulfillment-Location header.

Preferred P0: query param for GET cache visibility.

## GET /api/v1/commerce/categories?locationId=<id>

Returns only categories having at least one visible orderable product at the location.

## GET /api/v1/commerce/home?locationId=<id>

P0 home response is composed server-side:

~~~json
{
  "locationId": "store-location-id",
  "delivery": {
    "etaMinMinutes": 30,
    "etaMaxMinutes": 50
  },
  "sections": [
    {
      "key": "categories",
      "type": "CATEGORY_GRID",
      "title": "Shop by category",
      "items": []
    },
    {
      "key": "popular",
      "type": "PRODUCT_CAROUSEL",
      "title": "Popular in Birendranagar",
      "items": []
    },
    {
      "key": "daily-essentials",
      "type": "PRODUCT_CAROUSEL",
      "title": "Daily essentials",
      "items": []
    }
  ]
}
~~~

P0 section ranking may be manually configured in code/config. P1 moves section configuration to admin tables.

## GET /api/v1/commerce/products

Query:
- locationId required
- categoryId optional
- brandId optional
- productGroupId optional
- page default 1
- limit default 24 max 60
- sort optional: RELEVANCE, PRICE_ASC, PRICE_DESC, POPULAR

Product card response:

~~~json
{
  "id": "product-sku-id",
  "productGroupId": "group-id",
  "slug": "coca-cola-1l",
  "name": "Coca-Cola",
  "packLabel": "1 L",
  "brand": {
    "id": "brand-id",
    "name": "Coca-Cola"
  },
  "imageUrl": "https://...",
  "price": {
    "sellingPrice": 150,
    "mrp": 160,
    "discountPercent": 6
  },
  "availability": {
    "state": "AVAILABLE",
    "maxOrderQuantity": 8
  }
}
~~~

maxOrderQuantity may be capped by StoreProductConfig and does not need to reveal raw physical stock.

## GET /api/v1/commerce/products/:slug?locationId=<id>

PDP response includes:
- product group
- selected SKU
- all visible variants
- images
- description
- ingredients/nutrition/attributes if available
- availability
- price
- similar products
- frequently bought together placeholder

P0 similar products can use same category/brand rules.

---

# 21. Search API

## GET /api/v1/commerce/search

Query:
- q required
- locationId required
- page
- limit

P0 PostgreSQL search behavior:

Rank:
1. exact product name
2. exact brand + product tokens
3. prefix match
4. searchKeywords match
5. trigram similarity if pg_trgm is enabled
6. category match

P0 should normalize:
- lowercase
- whitespace
- punctuation
- common unit expressions
- configured aliases

Create SearchAlias in code/config first if avoiding another table.

Example aliases:
- dudh -> milk
- tel -> oil
- coke -> coca-cola
- surf -> surf excel
- diaper -> diapers

Response also includes:
- correctedQuery optional
- total
- products
- zeroResult boolean

P1 adds SearchEvent analytics and admin-managed aliases.

---

# 22. Cart API

## POST /api/v1/commerce/carts

No login required.

Request:

~~~json
{
  "inventoryLocationId": "store-location-id",
  "serviceZoneId": "zone-id"
}
~~~

Response includes cartToken.

Anonymous clients store cartToken locally.

## GET /api/v1/commerce/carts/:cartToken

Optional auth.

## POST /api/v1/commerce/carts/:cartToken/items

Request:

~~~json
{
  "productId": "sku-id",
  "unitId": "unit-id",
  "quantity": 1
}
~~~

Server:
1. validates cart active/not expired;
2. validates product visible at cart location;
3. validates unit belongs to product;
4. checks current availability;
5. applies max-order rule;
6. upserts CartItem;
7. recalculates cart.

## PATCH /api/v1/commerce/carts/:cartToken/items/:itemId

Request:

~~~json
{
  "quantity": 3
}
~~~

quantity 0 is not accepted here; use DELETE.

## DELETE /api/v1/commerce/carts/:cartToken/items/:itemId

## POST /api/v1/commerce/carts/:cartToken/revalidate

Reprices and rechecks all items.

Response:

~~~json
{
  "cart": {},
  "changes": [
    {
      "type": "PRICE_CHANGED",
      "productId": "..."
    }
  ],
  "checkoutAllowed": true
}
~~~

---

# 23. Checkout preview API

## POST /api/v1/commerce/checkout/preview

Auth required.

Request:

~~~json
{
  "cartToken": "cart-token",
  "addressId": "customer-address-id",
  "paymentMethod": "COD"
}
~~~

Server:
1. verifies customer owns address;
2. resolves zone from coordinates;
3. routes cart to a current eligible store;
4. if location changed, recalculates availability/price;
5. revalidates cart;
6. calculates min order / delivery fee;
7. returns final preview.

Response:

~~~json
{
  "fulfillmentLocationId": "store-location-id",
  "items": [],
  "subtotal": 1400,
  "discountTotal": 100,
  "deliveryFee": 60,
  "handlingFee": 0,
  "grandTotal": 1360,
  "etaMinMinutes": 30,
  "etaMaxMinutes": 50,
  "paymentMethod": "COD",
  "checkoutToken": "short-lived-signed-token"
}
~~~

checkoutToken expires quickly and binds:
- customer
- cart
- address
- routed location
- computed pricing version/hash

Place-order still revalidates everything. The token is not a bypass.

---

# 24. Place order API

## POST /api/v1/commerce/orders

Headers:

Idempotency-Key: required UUID

Auth required.

Request:

~~~json
{
  "cartToken": "cart-token",
  "addressId": "address-id",
  "paymentMethod": "COD",
  "checkoutToken": "...",
  "unavailableItemPreference": "REMOVE_ITEM"
}
~~~

P0 should reject partial fulfillment at placement despite the preference field. The field is retained for picking-stage shortage behavior.

Atomic server transaction:

1. validate Idempotency-Key using IdempotencyRecord scope commerce.order.create;
2. lock cart;
3. verify cart ACTIVE;
4. resolve customer/address;
5. re-run serviceability;
6. resolve complete-cart fulfillment location;
7. calculate all authoritative prices;
8. enforce minimum order;
9. create SalesOrder with source STOREFRONT, status PLACED, branchId, fulfillmentLocationId, customerId, immutable address snapshot;
10. create SalesOrderItem rows using server-calculated values;
11. create order-level StockReservation;
12. FEFO reserve all items from fulfillment location;
13. create OrderStatusEvent PLACED;
14. create CommercePayment with COD/PENDING for COD;
15. mark Cart CONVERTED and convertedOrderId;
16. complete IdempotencyRecord;
17. commit;
18. audit after commit using existing audit service;
19. return order.

If any inventory reservation fails, everything rolls back.

Response HTTP 201:

~~~json
{
  "id": "order-id",
  "orderNo": "ORD-...",
  "status": "PLACED",
  "grandTotal": 1360,
  "payment": {
    "method": "COD",
    "status": "PENDING"
  },
  "delivery": {
    "etaMinMinutes": 30,
    "etaMaxMinutes": 50
  }
}
~~~

Repeated identical Idempotency-Key:
- return same resource/result.

Same key with different request hash:
- 409 IDEMPOTENCY_KEY_REUSED.

No five-minute phone+total duplicate heuristic may remain.

---

# 25. Order read APIs

## GET /api/v1/commerce/orders

Auth required.

Pagination.

Only current customer's STOREFRONT orders.

## GET /api/v1/commerce/orders/:orderId

Customer ownership check mandatory.

Response includes:
- order summary
- items
- price snapshots
- delivery address snapshot
- payment
- status timeline

## GET /api/v1/commerce/orders/:orderId/tracking

Customer-friendly statuses only.

Internal statuses may be translated if needed.

---

# 26. Order cancellation

## POST /api/v1/commerce/orders/:orderId/cancel

Headers:
Idempotency-Key required.

Request:

~~~json
{
  "reason": "Changed my mind"
}
~~~

Customer may cancel P0 only from:
- PLACED
- CONFIRMED if that status is used before PICKING

Not cancelable after PICKING begins without staff workflow.

Transaction:
1. lock order;
2. validate owner;
3. validate state;
4. release order StockReservation RESERVED -> AVAILABLE;
5. set status CANCELLED;
6. cancelledAt;
7. create OrderStatusEvent;
8. update CommercePayment to CANCELLED if unpaid;
9. idempotency complete.

---

# 27. Staff fulfillment API

All staff fulfillment APIs require JwtAuthGuard + PermissionsGuard + ScopeGuard.

Base:
/api/v1/commerce/fulfillment

## GET /orders

Filters:
- branchId/locationId
- status
- page/limit

Default queues:
- PLACED
- PICKING
- PACKED
- OUT_FOR_DELIVERY

## POST /orders/:id/start-picking

Required:
- order is PLACED
- actor scoped to fulfillment branch/location

Transition:
PLACED -> PICKING

Set pickingStartedAt.

Extend reservation TTL or clear expiresAt while actively picking.

## POST /orders/:id/items/:orderItemId/pick

Request:
- pickedBaseQuantity
- barcode optional

P0 can persist pick progress in metadata or a dedicated FulfillmentPickItem model if partial picking visibility is required.

Recommended production model:

~~~prisma
model FulfillmentPickItem {
  id              String   @id @default(uuid())
  salesOrderItemId String
  pickedBaseQty   Decimal  @default(0) @db.Decimal(18, 6)
  shortageBaseQty Decimal  @default(0) @db.Decimal(18, 6)
  pickedById      String?
  updatedAt       DateTime @updatedAt

  salesOrderItem SalesOrderItem @relation(fields: [salesOrderItemId], references: [id], onDelete: Cascade)
  pickedBy       User?          @relation(fields: [pickedById], references: [id], onDelete: SetNull)

  @@unique([salesOrderItemId])
}
~~~

This model is P0 if staff picking UI is being built now.

## POST /orders/:id/report-shortage

Allowed actions P0:
- REMOVE_ITEM
- CONTACT_CUSTOMER

SUBSTITUTE is P1 unless explicitly implemented end-to-end.

A shortage must not directly mutate Product.stock.

## POST /orders/:id/pack

Requires all retained order items resolved.

Transition:
PICKING -> PACKED

Set packedAt.

## POST /orders/:id/out-for-delivery

Transition:
PACKED -> OUT_FOR_DELIVERY

Set outForDeliveryAt.

Create/update Delivery + DeliveryItem relation if delivery module is used.

At this transition, one of these must be selected and consistently implemented:

Preferred:
- create/post customer invoice sourced from fulfillment InventoryLocation;
- consume StockReservation using invoice reference;
- order inventory is now SOLD and no longer RESERVED.

Alternative:
- consume with SALES_ORDER reference at dispatch and create invoice asynchronously in same outbox workflow.

Do not leave inventory RESERVED after dispatch.

## POST /orders/:id/deliver

Transition:
OUT_FOR_DELIVERY -> DELIVERED

COD:
- record CommercePayment PAID;
- paidAt now;
- payment actor/receipt metadata;
- post required finance entries;
- set deliveredAt;
- status event.

If COD collection fails, do not mark DELIVERED as paid.

---

# 28. Order state machine

STOREFRONT transitions:

~~~text
PLACED
  -> PICKING
  -> PACKED
  -> OUT_FOR_DELIVERY
  -> DELIVERED

PLACED -> CANCELLED
PICKING -> PARTIALLY_FULFILLED only through shortage workflow
PACKED -> CANCELLED only staff override with reservation/accounting reversal
OUT_FOR_DELIVERY -> FAILED or DELIVERED
FAILED -> REFUND_PENDING when prepaid
REFUND_PENDING -> REFUNDED
~~~

Every transition uses OrderStateMachineService.

No controller/service may update SalesOrder.status directly except this service.

The state-machine service:
- validates current status;
- validates actor permissions/context;
- sets timestamps;
- creates OrderStatusEvent;
- performs transition-specific side effects in the same transaction where possible.

---

# 29. Reservation expiry worker

P0 add a scheduled job:

Every minute:
- find ACTIVE StockReservation with expiresAt < now;
- only release when order still PLACED and not accepted/picking;
- lock reservation and order;
- release RESERVED -> AVAILABLE;
- set order FAILED or CANCELLED with reason RESERVATION_EXPIRED depending product semantics;
- create status event.

Use @nestjs/schedule or existing worker infrastructure.

Do not release reservations for PICKING/PACKED orders.

---

# 30. Payment P0 and P1

## P0
COD only.

CommercePayment:
- method COD
- status PENDING at order placement
- PAID at confirmed delivery
- CANCELLED if order cancelled before collection

## P1
eSewa and Khalti.

Required payment pattern:
1. create payment intent server-side;
2. redirect/open provider;
3. provider callback/webhook;
4. server-to-server verification;
5. idempotent payment state transition;
6. never trust client success screen;
7. reconcile provider reference.

Add unique providerReference when safe.

---

# 31. Error contract

Continue using existing AppError codes where possible.

Add commerce-specific codes:

- CUSTOMER_AUTH_REQUIRED
- INVALID_OTP
- OTP_EXPIRED
- OUTSIDE_SERVICE_ZONE
- FULFILLMENT_LOCATION_UNAVAILABLE
- CART_NOT_FOUND
- CART_EXPIRED
- CART_LOCATION_CHANGED
- CART_PRICE_CHANGED
- CART_NOT_FULFILLABLE
- PRODUCT_NOT_ORDERABLE
- INSUFFICIENT_STOCK
- MIN_ORDER_NOT_MET
- ORDER_NOT_CANCELLABLE
- INVALID_ORDER_TRANSITION
- IDEMPOTENCY_KEY_REQUIRED
- IDEMPOTENCY_KEY_REUSED
- PAYMENT_FAILED

HTTP mapping:
- validation: 422
- unauthenticated: 401
- forbidden ownership/scope: 403
- not found: 404
- conflict/state/idempotency: 409
- provider temporarily unavailable: 503

---

# 32. NestJS module dependency graph

~~~text
CommerceModule
 |
 +-- CustomerAuthModule
 +-- CustomersModule
 +-- ServiceabilityModule
 +-- StorefrontCatalogModule
 |    +-- CatalogModule
 |    +-- InventoryModule
 +-- CartsModule
 |    +-- StorefrontCatalogModule
 +-- CheckoutModule
 |    +-- CartsModule
 |    +-- ServiceabilityModule
 |    +-- StorefrontCatalogModule
 |    +-- InventoryModule
 |    +-- SalesOrdersModule
 +-- FulfillmentModule
 |    +-- SalesOrdersModule
 |    +-- InventoryModule
 |    +-- DeliveryModule
 +-- CommercePaymentsModule
 +-- CommerceAnalyticsModule
~~~

Avoid circular imports by exposing narrow service interfaces from modules.

InventoryModule should export:
- StockReservationService
- InventorySnapshotService
- InventoryLedgerService

SalesOrdersModule should export only required order read/write abstraction, not raw Prisma.

Commerce services may still use PrismaService inside controlled transactions where cross-module atomicity is required.

---

# 33. AppModule change

Add:

~~~ts
import { CommerceModule } from "./commerce/commerce.module";

@Module({
  imports: [
    ...
    CommerceModule,
  ],
})
export class AppModule {}
~~~

Do not remove current SalesOrdersModule/CatalogModule/etc.

---

# 34. Replace createPublicOrder

The current SalesOrderService.createPublicOrder implementation must stop being the public checkout implementation.

Migration path:

1. keep endpoint temporarily behind compatibility flag;
2. change controller to delegate to CheckoutService or return 410 after clients migrate;
3. remove Product.stock checks;
4. remove notes JSON customer identity behavior;
5. remove five-minute duplicate heuristic;
6. after web/app cutover, delete or deprecate DTO and route.

No production web/app build may call the legacy public-order endpoint after P0 cutover.

---

# 35. Backend tests required before frontend integration

## Unit tests

ServiceabilityService:
- inside circle
- outside circle
- polygon boundary
- disabled zone
- disabled store mapping

AvailabilityService:
- available
- safety-stock subtraction
- expired batch excluded
- blocked batch excluded
- reserved stock not double-counted

PricingService:
- store override
- product fallback
- invalid/no price

CartService:
- add
- update
- remove
- expired cart
- location mismatch
- current price recalculation

StorefrontReservationService:
- single batch
- multi-batch FEFO
- rollback on insufficient stock
- concurrent oversell prevention
- release
- consume
- expired reservation release

CheckoutService:
- real idempotency
- duplicate key same hash
- duplicate key different hash
- cart repricing
- address ownership
- fulfillment reroute
- atomic rollback

OrderStateMachineService:
- all valid transitions
- all invalid transitions
- cancellation releases stock

## PostgreSQL integration tests

Must use PostgreSQL, not SQLite, because FOR UPDATE behavior matters.

Critical concurrency test:

Starting AVAILABLE=3.

Run two simultaneous checkouts:
- order A requests 2
- order B requests 2

Expected:
- one succeeds;
- one gets 422/409 stock conflict;
- AVAILABLE never negative;
- RESERVED total <= 3;
- ledger remains balanced.

---

# 36. API E2E acceptance suite

Required E2E flow:

1. create service zone and store location;
2. seed storefront product;
3. seed AVAILABLE inventory;
4. resolve serviceability;
5. fetch product;
6. create cart;
7. add product;
8. authenticate customer;
9. create address;
10. preview checkout;
11. place order with Idempotency-Key;
12. repeat same request/key and receive same order;
13. verify AVAILABLE decreased and RESERVED increased;
14. cancel order;
15. verify RESERVED decreased and AVAILABLE restored.

Second E2E flow:
- place order;
- start picking;
- pack;
- out for delivery;
- verify reservation consumed;
- deliver;
- verify COD paid;
- verify timeline.

---

# 37. Next.js 16 route specification

Target: modern-trade-website/frontend/src/app

Use App Router.

Required route tree:

~~~text
src/app/
  layout.tsx
  page.tsx                       /

  search/
    page.tsx                     /search?q=

  category/
    [slug]/
      page.tsx                   /category/dairy

  product/
    [slug]/
      page.tsx                   /product/amul-taaza-500ml

  cart/
    page.tsx                     /cart

  checkout/
    page.tsx                     /checkout

  order/
    success/
      [orderId]/
        page.tsx                 /order/success/:orderId

  orders/
    page.tsx                     /orders
    [orderId]/
      page.tsx                   /orders/:orderId

  account/
    page.tsx                     /account
    addresses/
      page.tsx                   /account/addresses
      new/
        page.tsx                 /account/addresses/new
      [addressId]/
        edit/
          page.tsx               /account/addresses/:id/edit

  login/
    page.tsx                     /login

  stores/
    page.tsx                     /stores

  about/
    page.tsx                     /about

  help/
    page.tsx                     /help

  privacy/
    page.tsx                     /privacy

  terms/
    page.tsx                     /terms

  api/
    auth/
      callback/
        route.ts                 only if needed for web session bridging
~~~

Do not add an internal ecommerce database under Next.js.

---

# 38. Next.js rendering rules

Server components:
- category page initial catalog
- product PDP
- SEO metadata
- public corporate pages

Client components:
- location selector
- search typeahead
- cart drawer/page
- quantity controls
- login OTP flow
- checkout interaction
- account/order actions

The site can use server-side fetches to Pasalho API for indexable content, but store/location selection must be respected.

Do not statically cache location-specific price/availability across users without a location cache key.

---

# 39. Next.js component structure

~~~text
src/components/commerce/
  header/
    CommerceHeader.tsx
    LocationSelector.tsx
    SearchBox.tsx
    CartButton.tsx

  category/
    CategoryGrid.tsx
    CategoryTile.tsx

  product/
    ProductCard.tsx
    ProductGrid.tsx
    ProductCarousel.tsx
    QuantityStepper.tsx
    VariantSelector.tsx
    PriceBlock.tsx
    AvailabilityBadge.tsx

  cart/
    CartDrawer.tsx
    CartLine.tsx
    CartSummary.tsx

  checkout/
    AddressSelector.tsx
    DeliverySummary.tsx
    PaymentMethodSelector.tsx
    PlaceOrderButton.tsx

  orders/
    OrderStatusTimeline.tsx
    OrderCard.tsx

  common/
    Money.tsx
    EmptyState.tsx
    Skeleton.tsx
    ErrorState.tsx
~~~

---

# 40. Next.js client state

P0:
- selected serviceability context persisted in cookie/localStorage;
- cartToken persisted locally;
- authenticated customer token strategy must be secure.

Preferred web auth:
- short-lived access token stored in secure HttpOnly same-site cookie through a Next.js BFF/session bridge, OR
- if calling API directly, use memory access token + HttpOnly refresh flow.

Do not store long-lived refresh tokens in localStorage.

Cart display may use React context/store, but the backend Cart is authoritative.

---

# 41. Web design behavior

Homepage order:

1. sticky header
2. delivery ETA/location
3. dominant search
4. campaign banner
5. categories
6. Buy Again when authenticated and data exists
7. Popular in Birendranagar
8. Daily Essentials
9. Fresh Vegetables
10. Dairy & Breakfast
11. Snacks & Drinks
12. Personal Care
13. Cleaning
14. Baby Care
15. Best Deals P1

Product cards:
- image
- name
- pack
- selling price
- MRP/discount
- ADD
- quantity stepper after add

Normal FMCG items must be addable without opening PDP.

---

# 42. Web SEO routes

Indexable:
- /
- /category/:slug
- /product/:slug
- /stores
- /about
- selected high-quality landing pages

Account/cart/checkout/order pages:
- noindex

Product page metadata must derive from canonical backend catalog.

Add Product structured data server-side using current live price/availability for a defined location only when semantically valid. If no location is known during crawler request, avoid pretending one store's inventory is globally available.

---

# 43. Flutter target navigation

Use go_router.

Route map:

~~~text
/
 /location
 /search
 /categories
 /category/:slug
 /product/:slug
 /cart
 /checkout
 /login
 /orders
 /orders/:orderId
 /account
 /account/addresses
 /account/addresses/new
 /account/addresses/:id/edit
 /loyalty       P1
 /help
~~~

Bottom navigation P0:

- Home
- Search
- Categories
- Orders
- Account

Cart is a persistent action/badge, not a bottom-tab replacement.

---

# 44. Flutter feature structure

Refactor from flat screens into feature modules incrementally:

~~~text
lib/
  app/
    app.dart
    router.dart
    theme.dart

  core/
    api/
      dio_client.dart
      api_error.dart
      auth_interceptor.dart
    storage/
      secure_token_store.dart
      local_store.dart
    models/
    utils/

  features/
    auth/
      data/
      domain/
      presentation/

    location/
      data/
      presentation/

    home/
      data/
      presentation/

    catalog/
      data/
      domain/
      presentation/

    search/
      data/
      presentation/

    cart/
      data/
      domain/
      presentation/

    checkout/
      data/
      presentation/

    orders/
      data/
      domain/
      presentation/

    account/
      data/
      presentation/

    loyalty/
      data/
      presentation/
~~~

Do not migrate every file in one commit. Move by feature while keeping builds green.

---

# 45. Flutter dependencies

P0 target additions:

~~~yaml
dependencies:
  flutter_riverpod: ^latest-compatible
  dio: ^latest-compatible
  go_router: ^latest-compatible
  flutter_secure_storage: existing
  shared_preferences: existing, for non-sensitive preferences
  intl: existing

dev_dependencies:
  riverpod_lint: compatible
~~~

If using Drift in P0:

~~~yaml
dependencies:
  drift: compatible
  sqlite3_flutter_libs: compatible
  path_provider: compatible
  path: compatible

dev_dependencies:
  drift_dev: compatible
  build_runner: compatible
~~~

Do not hard-code version guesses in implementation without checking current compatible versions when Codex performs the change.

Drift is optional for P0 unless offline catalog caching is being implemented. It is mandatory only when there is a defined offline requirement.

---

# 46. Flutter screens

## LocationScreen
Responsibilities:
- request location permission optionally;
- manual location/address search fallback;
- call serviceability resolve;
- persist shopping location;
- block shopping if unsupported.

## HomeScreen
Consumes GET /commerce/home.
No hard-coded fake products.

## SearchScreen
- debounce 250-350 ms;
- recent searches local;
- search suggestions P1;
- result product cards;
- zero-result Product Request P1.

## CategoryScreen
- query catalog by category slug/id;
- pagination/infinite scroll.

## ProductScreen
- group variants;
- selected SKU price and availability;
- add/stepper;
- product details.

## CartScreen
- server cart;
- optimistic quantity UX permitted;
- rollback on server rejection;
- revalidation banner when prices/availability change.

## LoginScreen
- phone -> OTP;
- secure tokens.

## CheckoutScreen
- address;
- order summary;
- COD P0;
- preview;
- idempotent place order;
- disable repeated submit while preserving same Idempotency-Key until definitive response.

## OrdersScreen
- customer order history.

## OrderDetailScreen
- customer status timeline;
- cancellation when eligible.

## AccountScreen
- profile;
- addresses;
- orders;
- loyalty P1;
- help.

---

# 47. Flutter Riverpod providers

Minimum:

- apiClientProvider
- authRepositoryProvider
- authStateProvider
- serviceabilityRepositoryProvider
- selectedLocationProvider
- homeRepositoryProvider
- homeProvider
- catalogRepositoryProvider
- categoryProductsProvider
- searchRepositoryProvider
- searchResultsProvider
- cartRepositoryProvider
- cartProvider
- checkoutRepositoryProvider
- checkoutPreviewProvider
- ordersRepositoryProvider
- ordersProvider
- orderDetailProvider

Avoid a single global AppState class containing all application behavior.

---

# 48. Flutter idempotency behavior

When user presses Place Order:

1. generate UUID idempotency key;
2. persist it in checkout state;
3. submit request;
4. if network timeout occurs before definitive server response, retry with the same key;
5. only generate a new key after confirmed failure that is safe to retry as a new logical operation;
6. on success clear checkout key and cart.

This rule prevents duplicate orders on poor mobile networks.

---

# 49. Analytics events

P0 backend/client event names:

- app_open
- home_view
- search
- search_result_click
- category_view
- product_view
- add_to_cart
- remove_from_cart
- cart_view
- checkout_started
- checkout_previewed
- order_created
- order_cancelled
- order_delivered

P0 may send them to existing analytics infrastructure.

P1 add CommerceEvent/SearchEvent persistence if internal demand analytics is required.

Never allow analytics failure to break checkout.

---

# 50. Caching

Redis may cache:
- category list by location
- storefront listing snapshots
- service-zone config
- home sections

Do not cache authoritative checkout stock.

Checkout/reservation must hit PostgreSQL transactionally.

Suggested TTL:
- categories: 5 minutes
- product card metadata: 1-5 minutes
- service-zone config: 1 minute or event-invalidated
- price: short TTL and revalidated at cart/checkout
- availability: very short or uncached for cart/checkout

Cache keys must include fulfillment location.

---

# 51. Security

Customer APIs:
- separate customer JWT guard from employee JwtAuthGuard;
- do not let customer tokens access internal admin controllers;
- ownership checks on addresses/carts/orders/payments;
- DTO whitelist already enabled globally;
- rate limits on OTP, search abuse, cart mutation and checkout;
- sanitize/validate address text;
- use signed URLs/object storage policies for uploads later.

Employee fulfillment:
- JwtAuthGuard
- PermissionsGuard
- ScopeGuard
- branch/location authorization

Required new permission codes:
- commerce.order.view
- commerce.order.pick
- commerce.order.pack
- commerce.order.dispatch
- commerce.order.deliver
- commerce.order.cancel
- commerce.payment.cod.confirm
- commerce.serviceability.manage
- commerce.storefront.manage

---

# 52. Configuration

Add to app config validation:

- CUSTOMER_JWT_SECRET
- CUSTOMER_JWT_EXPIRES_IN
- CUSTOMER_REFRESH_TOKEN_DAYS
- OTP_PROVIDER
- OTP_TTL_SECONDS
- OTP_MAX_ATTEMPTS
- STOREFRONT_SYSTEM_USER_ID or system-user lookup key
- CART_TTL_HOURS
- ORDER_RESERVATION_TTL_MINUTES
- STOREFRONT_DEFAULT_CURRENCY=NPR
- COMMERCE_CORS_ORIGINS if separated
- ESEWA_* P1
- KHALTI_* P1

Do not hard-code secret/provider credentials.

---

# 53. Observability

Structured log fields for commerce:
- requestId
- customerId
- cartId
- salesOrderId
- fulfillmentLocationId
- idempotencyKey
- paymentId
- errorCode

Metrics:
- checkout attempts
- checkout success
- reservation failures
- stock conflicts
- idempotent replays
- order status transition latency
- picking duration
- packing duration
- delivery duration
- cancellation rate
- out-of-stock search/cart events

Never log OTP codes, JWTs, refresh tokens or complete provider credentials.

---

# 54. P0 engineering sequence

P0 is launch-critical. Do not start with homepage styling.

## P0-01 Current-state guardrails

Backend:
- add tests proving storefront code must not use Product.stock;
- configure system actor lookup;
- document current endpoint deprecation;
- ensure ScopeGuard remains enabled for staff commerce.

Acceptance:
- existing tests pass;
- grep/static check identifies createPublicOrder legacy stock usage before replacement.

Commit:
chore(commerce): establish storefront guardrails

## P0-02 Customer identity + address

Implement Migration 1.
Implement CustomerAuthModule and CustomersModule.

Acceptance:
- OTP adapter interface;
- dev/test provider;
- verify OTP creates customer;
- refresh rotation;
- address CRUD ownership tests.

Commit:
feat(commerce): add customer auth and addresses

## P0-03 Serviceability + fulfillment routing

Implement ServiceZone models, resolver and routing.

Seed:
- one Birendranagar pilot zone;
- explicit mapped STORE InventoryLocation.

Acceptance:
- outside zone rejected;
- disabled store rejected;
- deterministic routing.

Commit:
feat(commerce): add serviceability and fulfillment routing

## P0-04 Storefront catalog + variants + location availability

Implement Migration 2.
Implement StorefrontCatalogModule, AvailabilityService, PricingService.

Acceptance:
- product cards derive availability from InventorySnapshot;
- no Product.stock;
- store-specific visibility and price supported;
- variant grouping works.

Commit:
feat(commerce): add store-scoped catalog and pricing

## P0-05 Inventory reservation hardening

Implement Migration 3.
Add StorefrontReservationService.

Required:
- FEFO multi-batch reservation;
- StockReservation/Item attribution;
- release;
- consume;
- concurrency tests.

Acceptance:
- no negative stock under concurrent checkout;
- transaction rollback is complete.

Commit:
feat(inventory): harden storefront reservations

## P0-06 Server cart

Implement Cart/CartItem and Migration 4 cart portion.

Acceptance:
- anonymous cart token;
- add/update/remove;
- location scoped;
- price and stock revalidation;
- TTL.

Commit:
feat(commerce): add server-authoritative cart

## P0-07 Checkout + order placement

Extend SalesOrder via Migration 4.
Implement preview/place order.
Replace createPublicOrder usage.

Acceptance:
- Idempotency-Key required;
- same key returns same order;
- different request same key 409;
- checkout creates reservation atomically;
- immutable address snapshot;
- correct branch/location on order.

Commit:
feat(commerce): add atomic storefront checkout

## P0-08 Order state machine + cancellation

Implement OrderStatusEvent.
Implement OrderStateMachineService.
Implement customer read/cancel endpoints.
Implement reservation expiry worker.

Acceptance:
- invalid transitions impossible;
- cancellation restores AVAILABLE;
- timeline always written.

Commit:
feat(commerce): add storefront order lifecycle

## P0-09 Staff picking/packing

Implement fulfillment queue and pick workflow.
Add FulfillmentPickItem model if required.

Acceptance:
- staff cannot act outside scope;
- picking cannot exceed ordered quantity;
- shortage path explicit;
- pack only when items resolved.

Commit:
feat(commerce): add store fulfillment workflow

## P0-10 COD + dispatch + inventory consumption

Implement Migration 5.
Implement CommercePayment COD.
Make Invoice warehouse optional if invoice path is selected.
Extend DeliveryItem order relation.

Acceptance:
- reservation consumed exactly once;
- dispatch cannot double-consume;
- COD marked paid only on delivery;
- cancellation/reversal paths tested.

Commit:
feat(commerce): finalize COD fulfillment

## P0-11 Next.js Blinkit-style commerce surface

In modern-trade-website:
- route tree above;
- location selector;
- search-first header;
- home;
- product cards with inline ADD;
- category;
- PDP;
- cart;
- login;
- checkout;
- orders/account;
- mobile responsive.

Acceptance:
- uses commerce API only;
- no fake inventory;
- no client price truth;
- Playwright purchase flow;
- mobile 360px layout.

Commit:
feat(storefront): rebuild customer web commerce

## P0-12 Flutter commerce refactor

In modern-trade-app:
- add Riverpod/Dio/go_router;
- refactor auth/location/catalog/cart/checkout/orders;
- preserve app build;
- connect exact same API.

Acceptance:
- Android build;
- Codemagic build config still works;
- place order idempotency survives retry;
- no duplicated pricing/stock logic.

Commit:
feat(app): integrate Pasalho commerce API

## P0-13 End-to-end release gate

Run:
- prisma validate
- prisma migrate deploy on staging copy
- backend unit/integration/e2e
- web lint/typecheck/build/Playwright
- Flutter analyze/test/build
- concurrency checkout test
- cancellation restoration test
- full fulfillment test

Release only when all critical acceptance checks pass.

---

# 55. P1 engineering sequence

P1 starts only after P0 is stable in controlled beta.

## P1-01 eSewa + Khalti
- provider adapters
- callbacks/webhooks
- server verification
- payment reconciliation
- refunds

## P1-02 Buy Again
- order-history derived
- current price/availability on reorder
- never reuse historical price blindly

## P1-03 Promotions and coupons
- Promotion model
- rule engine
- server-side discount
- coupon usage limits
- audit

## P1-04 Loyalty
- LoyaltyAccount/LoyaltyEntry
- earn on delivered/paid order
- redemption transactionally reserved/applied
- reversal on refund

## P1-05 Push notifications
- CustomerDevice
- order status notifications
- campaign opt-in controls

## P1-06 Search quality
- pg_trgm or full-text
- aliases
- zero-result analytics
- product request
- popularity ranking

## P1-07 Recommendations
Start rules-based:
- same category
- complementary mapping
- buy again
- order co-occurrence only after enough data

## P1-08 Supplier-funded merchandising
- banners
- sponsored search/category positions
- explicit Sponsored label
- campaign attribution

## P1-09 Delivery automation
- rider entities/assignment
- capacity
- ETA
- live tracking only when operationally justified

## P1-10 Advanced experimentation
- feature flags
- A/B tests
- conversion funnel metrics

---

# 56. Exact web acceptance journey

A first-time web customer must be able to:

1. open /;
2. choose a serviceable Birendranagar location;
3. search "milk";
4. add from product card;
5. add another category product;
6. open cart;
7. authenticate via phone OTP;
8. select/create address;
9. preview checkout;
10. place COD order;
11. see order success;
12. view order tracking;
13. cancel while PLACED;
14. see inventory become sellable again.

No full page should expose an internal exception stack.

---

# 57. Exact app acceptance journey

A mobile customer must be able to:

1. launch;
2. restore token/location/cart;
3. browse without login;
4. add/remove items;
5. login only when required;
6. place COD order with idempotency;
7. kill/reopen app after network timeout;
8. retry without duplicate order;
9. see order history and status;
10. reorder P1.

---

# 58. Performance targets

Backend:
- catalog GET p95 < 500 ms under pilot load
- search p95 < 700 ms
- cart mutation p95 < 500 ms
- checkout placement p95 < 2 s excluding external payment provider
- no N+1 product card query pattern

Web:
- useful mobile content < 2.5 s on normal Nepal 4G target
- responsive at 360px
- image optimization
- skeletons
- route-level loading states

App:
- home first usable content quickly from network/cache
- no UI blocking on analytics
- resilient to intermittent network

---

# 59. Data seeding required before UI QA

Create a pilot commerce seed script that does not overwrite production master data.

Seed or map:
- Birendranagar service zone
- one STORE InventoryLocation
- service-zone mapping
- 8-12 categories
- representative ProductGroups
- 50-100 visible StoreProductConfig rows
- realistic InventorySnapshot AVAILABLE stock
- product images
- system actor user
- COD configuration

Do not rely on Product.stock seed values.

---

# 60. CI gates

Backend:
- npm test
- PostgreSQL integration tests
- prisma validate
- prisma migrate diff/check as appropriate
- build
- lint if configured

Web:
- npm run lint
- npm run type-check
- npm run build
- npm run test:e2e:chromium
- accessibility smoke
- SEO smoke

Flutter:
- flutter analyze
- flutter test
- flutter build apk --debug or CI-equivalent
- Codemagic workflow syntax validation

No P0 task is complete if its repository does not build.

---

# 61. Rollout

Stage 0:
- local + seeded PostgreSQL

Stage 1:
- internal staff
- one store/location
- test orders only

Stage 2:
- 50-100 controlled customers
- small Birendranagar zone
- COD only

Stage 3:
- expand zone after:
  - fulfillment accuracy acceptable
  - stock conflicts low
  - cancellation reasons understood
  - delivery economics measured

Stage 4:
- add second store and test routing

Only after multi-store routing is proven should the homepage promise broader hyperlocal coverage.

---

# 62. Prohibited shortcuts

Codex must not:

- read or mutate Product.stock for storefront;
- deduct AVAILABLE stock without reservation;
- create a second ecommerce inventory table as source of truth;
- let frontend send authoritative price;
- let frontend choose an unvalidated fulfillment store;
- store customer address only in notes;
- use phone+total+time as idempotency;
- create separate product databases in web and app;
- create separate APIs for web and Flutter for the same commerce action;
- update order status directly outside OrderStateMachineService;
- skip PostgreSQL concurrency tests;
- expose employee JWT auth endpoints to customer tokens;
- hard-code OTP, payment, JWT or provider secrets;
- promise 8-10 minute delivery in code/UI unless operations later approve it.

---

# 63. Codex execution protocol

Codex should implement one P0 item at a time.

For every item:

1. inspect current repository state first;
2. do not discard unrelated user changes;
3. make the smallest coherent change;
4. add/update tests;
5. run relevant validation;
6. report exact files changed;
7. report migrations created;
8. report tests run and result;
9. report unresolved risks;
10. commit only requested work with a focused commit message.

Do not execute multiple migration phases in one uncontrolled commit.

---

# 64. First Codex task

Start with P0-01 through P0-03 only if the user asks for an initial implementation batch.

Suggested instruction:

"Read TECHNICAL_IMPLEMENTATION_SPEC.md in surkhettimes05-boop/Blinkit-style-pasal-ho. Inspect the current main branch of surkhettimes05-boop/pasalho before changing code. Implement P0-01, P0-02 and P0-03 exactly, preserving unrelated work. Use additive Prisma migrations. Do not touch Product.stock except to mark/deprecate storefront usage. Add tests, run prisma validate, PostgreSQL-backed tests relevant to the change, and backend build. Stop after P0-03 and report commit hashes, files changed, migration names, test results and blockers."

For the safest execution, P0-04/P0-05 should be a separate Codex run because inventory semantics are high risk.

---

# 65. Definition of P0 done

P0 is done only when:

- location resolves to a valid fulfillment store;
- catalog is store-scoped;
- customer-facing stock comes from InventorySnapshot;
- cart is server-backed;
- checkout is server-priced;
- order creation is idempotent;
- inventory is reserved atomically;
- concurrent checkout cannot oversell;
- customer orders are attributed to Customer + address snapshot;
- picking/packing workflow exists;
- dispatch consumes reservation;
- delivery closes COD correctly;
- cancellation releases stock;
- web and app use the same API;
- Next.js builds and E2E flow passes;
- Flutter builds and checkout flow passes;
- no storefront production path reads Product.stock.

---

# 66. Final architecture contract

There is one source of truth for each concern:

- Product master: Pasalho Product
- Variant grouping: ProductGroup
- Store visibility/price policy: StoreProductConfig
- Physical/sellable inventory: InventorySnapshot + InventoryEvent/Movement
- Reservation attribution: StockReservation + StockReservationItem
- Customer order: SalesOrder with source STOREFRONT
- Customer identity: Customer
- Cart: Cart
- Payment: CommercePayment
- Fulfillment location: InventoryLocation
- Delivery lifecycle: SalesOrder status + Delivery
- Web/mobile APIs: /api/v1/commerce

The Blinkit-style UI is the surface.

The real product is the integration of search, pricing, store inventory, checkout, reservation and fulfillment into one reliable system.
