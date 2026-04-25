# Zero-Commission Food Delivery Platform — Build Spec

> **AI AGENT INSTRUCTIONS**: Read this file in full before writing any code.
> All field names, enum values, money handling, and flow logic defined here are CANONICAL.
> Do not invent alternatives. If something is ambiguous, ask before implementing.

---

## Project Overview

A subscription-based food delivery platform with 4 systems:
- **Customer App** (React Native / Expo)
- **Driver App** (React Native / Expo)
- **Restaurant Dashboard** (React web)
- **Admin Panel** (React web)

**Core business rules (never violate these):**
- Drivers pay a monthly subscription to access orders
- Restaurants pay a monthly subscription to receive orders
- Platform takes ZERO commission per order
- Drivers keep 100% of delivery fees + tips
- Restaurants keep 100% of food revenue
- Customer pays ONE combined charge (food + delivery fee + tip)
- All money values are stored and calculated in **cents (integers), never floats**

---

## Tech Stack (Exact)

| Layer | Tool |
|---|---|
| Mobile apps (customer + driver) | React Native + Expo (managed workflow) |
| Web apps (restaurant + admin) | React + Vite |
| State management | Zustand |
| API server | Node.js + Fastify |
| Real-time | Socket.io |
| Auth | JWT + refresh tokens (bcrypt for passwords) |
| Background jobs | BullMQ (Redis-backed) |
| ORM | Prisma |
| Primary database | PostgreSQL |
| Cache + geo queries | Redis |
| File storage | Cloudflare R2 (presigned URLs) |
| Payments | Stripe (Connect + Subscriptions + Payment Intents) |
| Push notifications | Firebase Cloud Messaging (FCM) |
| Maps + distance | Google Maps Platform (Places, Distance Matrix, Directions) |
| Email | Resend |
| SMS / OTP | Twilio Verify |
| Error monitoring | Sentry |
| Analytics | PostHog |
| Mobile builds | EAS Build (Expo) |
| API hosting | Railway or Render |
| Web hosting | Vercel |

**Monorepo structure (Turborepo):**
```
/apps
  /customer        # React Native (Expo)
  /driver          # React Native (Expo)
  /restaurant      # React + Vite
  /admin           # React + Vite
/packages
  /api             # Fastify server
  /shared          # Shared types, utils, constants
```

---

## Database Schema (Prisma — Canonical)

> Use these EXACT field names everywhere: in API routes, frontend state, and Stripe metadata.

### users
```prisma
model User {
  id                String    @id @default(uuid())
  email             String    @unique
  phone             String    @unique
  role              UserRole
  fullName          String
  passwordHash      String
  stripeCustomerId  String?
  stripeAccountId   String?   // Stripe Connect account (drivers + restaurants only)
  status            UserStatus @default(PENDING_VERIFICATION)
  fcmToken          String?
  createdAt         DateTime  @default(now())

  driverProfile     DriverProfile?
  restaurantProfile RestaurantProfile?
  subscription      Subscription?
  ordersAsCustomer  Order[]   @relation("CustomerOrders")
  ratings           Rating[]
}

enum UserRole {
  CUSTOMER
  DRIVER
  RESTAURANT
  ADMIN
}

enum UserStatus {
  ACTIVE
  SUSPENDED
  BANNED
  PENDING_VERIFICATION
}
```

### driver_profiles
```prisma
model DriverProfile {
  userId              String   @id
  user                User     @relation(fields: [userId], references: [id])
  baseFee             Int      // cents, e.g. 300 = $3.00
  perMileFee          Int      // cents per mile, e.g. 100 = $1.00/mi
  minimumPayout       Int      // cents, floor per delivery
  isOnline            Boolean  @default(false)
  currentLat          Float?
  currentLng          Float?
  rating              Decimal  @default(5.0) @db.Decimal(3, 2)
  totalDeliveries     Int      @default(0)
  vehicleType         VehicleType
  licenseNumber       String

  ordersAsDriver      Order[]  @relation("DriverOrders")
}

enum VehicleType {
  CAR
  BIKE
  SCOOTER
}
```

### restaurant_profiles
```prisma
model RestaurantProfile {
  userId          String   @id
  user            User     @relation(fields: [userId], references: [id])
  businessName    String
  address         String
  lat             Float
  lng             Float
  cuisineType     String   // comma-separated tags
  isOpen          Boolean  @default(false)
  operatingHours  Json     // { "mon": ["09:00","22:00"], "tue": [...], ... }
  rating          Decimal  @default(5.0) @db.Decimal(3, 2)
  taxId           String

  menuCategories  MenuCategory[]
  orders          Order[]  @relation("RestaurantOrders")
}
```

### menu_categories
```prisma
model MenuCategory {
  id            String   @id @default(uuid())
  restaurantId  String
  restaurant    RestaurantProfile @relation(fields: [restaurantId], references: [userId])
  name          String
  sortOrder     Int      @default(0)
  items         MenuItem[]
}
```

### menu_items
```prisma
model MenuItem {
  id            String   @id @default(uuid())
  categoryId    String
  category      MenuCategory @relation(fields: [categoryId], references: [id])
  name          String
  description   String?
  priceCents    Int
  photoUrl      String?
  isAvailable   Boolean  @default(true)
}
```

### orders
```prisma
model Order {
  id                  String      @id @default(uuid())
  customerId          String
  customer            User        @relation("CustomerOrders", fields: [customerId], references: [id])
  restaurantId        String
  restaurant          RestaurantProfile @relation("RestaurantOrders", fields: [restaurantId], references: [userId])
  driverId            String?
  driver              DriverProfile? @relation("DriverOrders", fields: [driverId], references: [userId])
  status              OrderStatus @default(PENDING)
  items               Json        // Snapshot at order time: [{name, priceCents, qty, menuItemId}]
  subtotalCents       Int         // Food only
  deliveryFeeCents    Int         // Driver-calculated at order time
  tipCents            Int         @default(0)
  totalCents          Int         // subtotal + deliveryFee + tip + stripeFee
  deliveryAddress     Json        // Snapshot: {line1, city, state, zip, lat, lng}
  specialInstructions String?
  createdAt           DateTime    @default(now())
  deliveredAt         DateTime?

  payment             Payment?
  rating              Rating?
}

enum OrderStatus {
  PENDING                // Placed, awaiting restaurant acceptance
  RESTAURANT_ACCEPTED    // Restaurant confirmed
  DRIVER_ASSIGNED        // Driver accepted
  PICKED_UP              // Driver picked up food
  DELIVERED              // Completed
  CANCELED               // Canceled at any stage
  REFUNDED               // Refund issued
}
```

### payments
```prisma
model Payment {
  id                      String        @id @default(uuid())
  orderId                 String        @unique
  order                   Order         @relation(fields: [orderId], references: [id])
  stripePaymentIntentId   String
  restaurantTransferId    String?       // Filled after delivery
  driverTransferId        String?       // Filled after delivery
  status                  PaymentStatus @default(PENDING)
  restaurantPayoutCents   Int
  driverPayoutCents       Int
  refundAmountCents       Int           @default(0)
}

enum PaymentStatus {
  PENDING
  CAPTURED
  TRANSFERRED
  REFUNDED
  PARTIALLY_REFUNDED
}
```

### subscriptions
```prisma
model Subscription {
  id                    String             @id @default(uuid())
  userId                String             @unique
  user                  User               @relation(fields: [userId], references: [id])
  stripeSubscriptionId  String
  plan                  SubscriptionPlan
  status                SubscriptionStatus @default(TRIALING)
  currentPeriodEnd      DateTime
}

enum SubscriptionPlan {
  DRIVER_BASIC     // $29/month
  DRIVER_PRO       // $49/month
  RESTAURANT_STARTER  // $99/month
  RESTAURANT_GROWTH   // $199/month
}

enum SubscriptionStatus {
  ACTIVE
  TRIALING
  PAST_DUE
  CANCELED
}
```

### ratings
```prisma
model Rating {
  id           String   @id @default(uuid())
  orderId      String   @unique
  order        Order    @relation(fields: [orderId], references: [id])
  customerId   String
  customer     User     @relation(fields: [customerId], references: [id])
  driverRating Int      // 1–5
  foodRating   Int      // 1–5
  reviewText   String?
  createdAt    DateTime @default(now())
}
```

---

## Order State Machine

Valid transitions only (enforce server-side, reject all others):

```
PENDING → RESTAURANT_ACCEPTED    (trigger: restaurant accepts, within 3 min)
PENDING → CANCELED               (trigger: 3-min timeout, or customer cancels)
RESTAURANT_ACCEPTED → DRIVER_ASSIGNED  (trigger: driver accepts dispatch)
RESTAURANT_ACCEPTED → CANCELED   (trigger: no driver found after 3 dispatch rounds)
DRIVER_ASSIGNED → PICKED_UP      (trigger: driver taps "Picked up")
DRIVER_ASSIGNED → CANCELED       (trigger: driver cancels — reassign attempted first)
PICKED_UP → DELIVERED            (trigger: driver taps "Delivered" → fires payouts)
DELIVERED → REFUNDED             (trigger: admin initiates refund)
CANCELED → REFUNDED              (trigger: automatic on payment-captured orders)
```

**On DELIVERED — fire immediately (same DB transaction):**
1. Set `order.deliveredAt = now()`
2. Set `payment.status = TRANSFERRED`
3. `stripe.transfers.create` → restaurant (`payment.restaurantPayoutCents`)
4. `stripe.transfers.create` → driver (`payment.driverPayoutCents` + `order.tipCents`)
5. Store both transfer IDs on `payment` record
6. Push notification to customer: "Your order has arrived"

---

## Payment & Payout Logic

### At checkout (order creation):
```javascript
// 1. Calculate delivery fee (server-side only, never trust client)
const distanceMiles = await googleMaps.distanceMatrix(restaurantCoords, customerCoords);
const deliveryFee = Math.max(
  driver.minimumPayout,
  driver.baseFee + Math.round(distanceMiles * driver.perMileFee)
); // All in cents

// 2. Calculate Stripe fee to pass through (optional — add to total)
const stripeFee = Math.round(total * 0.029) + 30; // cents

// 3. Create Payment Intent
const paymentIntent = await stripe.paymentIntents.create({
  amount: subtotal + deliveryFee + tip + stripeFee,
  currency: 'usd',
  customer: user.stripeCustomerId,
  transfer_group: orderId,
  metadata: { orderId, restaurantId, driverId }
});

// 4. Store payout amounts on payment record (locked at order time)
payment.restaurantPayoutCents = subtotal;
payment.driverPayoutCents = deliveryFee; // tip transferred separately
```

### Payout split:
| Component | Recipient | Transfer timing |
|---|---|---|
| Food subtotal | Restaurant Connected Account | On DELIVERED |
| Delivery fee | Driver Connected Account | On DELIVERED |
| Tip | Driver Connected Account | On DELIVERED (or after 15-min adjust window) |
| Platform revenue | None — platform keeps $0 of order value | — |

### Refund matrix:
| Scenario | Refund | Payout reversal |
|---|---|---|
| Restaurant no-accept (timeout) | Full refund | No transfer made — nothing to reverse |
| No driver found | Full refund | No transfer made |
| Driver cancels after pickup | Full refund | `stripe.transfers.createReversal` on restaurant transfer |
| Wrong item / quality | Partial or full (admin) | Partial reversal on restaurant transfer |
| Customer no-show | No refund | Transfers complete normally |

---

## Stripe Webhooks (Implement All 7)

```
payment_intent.succeeded        → Mark payment CAPTURED, create order record
payment_intent.payment_failed   → Log failure, notify customer, do NOT create order
customer.subscription.created   → Set subscription.status = ACTIVE, activate user
customer.subscription.past_due  → Set subscription.status = PAST_DUE, remove from dispatch pool
customer.subscription.deleted   → Set subscription.status = CANCELED, hard-remove access
transfer.failed                 → Alert admin, add to failed-transfers queue for manual retry
account.updated                 → Sync Connect account verification status on user record
```

Always verify webhook signature:
```javascript
const event = stripe.webhooks.constructEvent(payload, sig, process.env.STRIPE_WEBHOOK_SECRET);
```

---

## Dispatch System

When `order.status` transitions to `RESTAURANT_ACCEPTED`:

**Round 1 (0–5 miles):**
1. Query Redis GEORADIUS: drivers within 5mi of restaurant, `isOnline = true`, `subscription.status = ACTIVE`, no active order
2. Score each: `score = (1/distance_km * 0.5) + (rating/5 * 0.3) + (normalizedDeliveries * 0.2)`
3. Sort descending, take top 5
4. Push notification to all 5 simultaneously via FCM
5. Start 45-second BullMQ timer

**If no acceptance after 45s:**
- Round 2: expand to 5–7mi, repeat
- Round 3: expand to 7–9mi, repeat
- After Round 3: notify customer "No driver available — cancel for full refund or wait"
- Wait option: re-enters dispatch queue every 2 minutes

**Driver location updates:**
- Online drivers send GPS every 10 seconds via WebSocket
- Store in Redis: `GEOADD drivers:online lng lat userId`
- Sync to PostgreSQL `driverProfile.currentLat/Lng` every 30s (not real-time)

---

## Subscription Enforcement

Add middleware to all dispatch and order-receiving routes:

```javascript
// Driver routes (access orders):
if (driver.subscription.status !== 'ACTIVE' && driver.subscription.status !== 'TRIALING') {
  return reply.status(403).json({ error: 'SUBSCRIPTION_REQUIRED' });
}

// Restaurant routes (receive orders):
if (restaurant.subscription.status !== 'ACTIVE' && restaurant.subscription.status !== 'TRIALING') {
  return reply.status(403).json({ error: 'SUBSCRIPTION_REQUIRED' });
}
```

Grace period: 3 days after `PAST_DUE` before access is suspended (configurable).

---

## API Route Structure

```
POST   /auth/signup
POST   /auth/login
POST   /auth/refresh

GET    /restaurants                    # Browse (customer)
GET    /restaurants/:id/menu           # Menu (customer)

POST   /orders                         # Create order + charge (customer)
GET    /orders/:id                     # Order detail
PATCH  /orders/:id/status              # Status transition (role-restricted)
GET    /orders/customer/history        # Customer order history

POST   /drivers/online                 # Go online (driver)
POST   /drivers/offline                # Go offline (driver)
PATCH  /drivers/pricing                # Update base fee, per-mile, minimum
GET    /drivers/earnings               # Earnings summary

GET    /restaurant/orders              # Incoming + active orders (restaurant)
POST   /restaurant/menu/categories     # Create category
POST   /restaurant/menu/items          # Create item
PATCH  /restaurant/menu/items/:id      # Update item (price, availability)

POST   /subscriptions/create           # Create Stripe subscription
DELETE /subscriptions/cancel           # Cancel subscription
POST   /webhooks/stripe                # Stripe webhook handler

GET    /admin/users                    # All users
PATCH  /admin/users/:id/status         # Suspend/ban
GET    /admin/orders                   # Full order log
POST   /admin/orders/:id/refund        # Issue refund
PATCH  /admin/subscriptions/:id        # Override subscription
```

---

## Real-time Socket.io Events

```javascript
// Customer joins order room on order placement
socket.join(`order:${orderId}`);

// Events emitted to order room:
'order:status_changed'    // { orderId, status, timestamp }
'driver:location_update'  // { orderId, lat, lng }
'order:driver_assigned'   // { orderId, driverName, vehicleType }

// Driver events:
'dispatch:new_order'      // { orderId, restaurantName, restaurantAddress, estimatedPayout, distanceMiles }
'dispatch:cancelled'      // { orderId } — order taken by another driver

// Restaurant events:
'restaurant:new_order'    // { orderId, items, subtotal, specialInstructions }
```

---

## Environment Variables Required

```env
# Database
DATABASE_URL=

# Redis
REDIS_URL=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PUBLISHABLE_KEY=

# Google Maps
GOOGLE_MAPS_API_KEY=

# Firebase
FIREBASE_PROJECT_ID=
FIREBASE_PRIVATE_KEY=
FIREBASE_CLIENT_EMAIL=

# Cloudflare R2
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=

# Twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_VERIFY_SERVICE_SID=

# Resend
RESEND_API_KEY=

# App
JWT_SECRET=
JWT_REFRESH_SECRET=
NODE_ENV=development
PORT=3000
```

---

## Critical Implementation Rules

1. **All money in cents.** Never store or calculate dollars. `$3.50` = `350`. Display only converts to dollars.
2. **Snapshot order items.** Copy item name + priceCents into `order.items` JSON at order creation. Never join back to live menu for order history.
3. **Server-side delivery fee.** Never trust the client's delivery fee calculation. Always recalculate on the server using Google Distance Matrix.
4. **Stripe webhook idempotency.** Check if event has already been processed before acting. Store processed event IDs.
5. **Transfer group.** Always pass `transfer_group: orderId` to PaymentIntent and both Transfers for reconciliation.
6. **Role-based middleware.** Every route must declare which roles can access it. Enforce at the middleware level, not inside route handlers.
7. **Order item snapshot format:** `[{ menuItemId, name, priceCents, qty, modifiers: [] }]`
8. **Never expose `passwordHash`, `stripeCustomerId`, or `stripeAccountId` in API responses to clients.**
