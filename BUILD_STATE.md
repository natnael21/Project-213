# Build State — Zero-Commission Food Delivery Platform

> **HOW TO USE THIS FILE**
> Update this file at the end of every session.
> Start every new session with: `@SPEC.md @BUILD_STATE.md Continue from where we left off. Next task is [CURRENT TASK].`

---

## Current Status
**Phase:** 1 — Foundation
**Current task:** 1.2 Database setup
**Last updated:** 2026-04-20

---

## Phase 1 — Foundation (Weeks 1–2)

### 1.1 Project scaffold
- [x] Turborepo monorepo initialized
- [x] `/apps/customer` (React Native + Expo)
- [x] `/apps/driver` (React Native + Expo)
- [x] `/apps/restaurant` (React + Vite)
- [x] `/apps/admin` (React + Vite)
- [x] `/packages/api` (Fastify)
- [x] `/packages/shared` (types, utils, constants)
- [x] ESLint + Prettier configured
- [x] Environment variable files (`.env.example` for each app)

### 1.2 Database
- [ ] Prisma initialized in `/packages/api`
- [ ] `schema.prisma` — all models from SPEC.md Section: Database Schema
- [ ] All enums defined (UserRole, UserStatus, OrderStatus, PaymentStatus, SubscriptionPlan, SubscriptionStatus, VehicleType)
- [ ] First migration run (`prisma migrate dev`)
- [ ] Seed file with test data (1 admin, 1 customer, 1 driver, 1 restaurant)

### 1.3 Auth
- [ ] `POST /auth/signup` — all roles
- [ ] `POST /auth/login` — returns access + refresh token
- [ ] `POST /auth/refresh` — rotates refresh token
- [ ] Role-based middleware (`requireRole(UserRole[])`)
- [ ] JWT secret + refresh secret in env vars
- [ ] Passwords hashed with bcrypt (rounds: 12)

### 1.4 Stripe setup
- [ ] Stripe SDK installed, initialized with `STRIPE_SECRET_KEY`
- [ ] Webhook endpoint `POST /webhooks/stripe` created
- [ ] Webhook signature verification implemented
- [ ] All 7 webhooks handled (see SPEC.md: Stripe Webhooks)
- [ ] Stripe utility functions: createCustomer, createConnectAccountLink, createSubscription, createPaymentIntent, createTransfer, createRefund, createTransferReversal
- [ ] Webhook idempotency: processed event IDs stored

### 1.5 Firebase / FCM
- [ ] Firebase Admin SDK initialized
- [ ] `fcmToken` stored on user record at login
- [ ] `sendPushNotification(userId, title, body, data)` utility function

### 1.6 Redis
- [ ] Redis connected (Upstash or local)
- [ ] BullMQ initialized with Redis connection
- [ ] `dispatch` queue defined
- [ ] Driver geo index: `GEOADD drivers:online` working

---

## Phase 2 — Core Ordering (Weeks 3–4)

### 2.1 Restaurant dashboard — menu builder
- [ ] `POST /restaurant/menu/categories` — create category
- [ ] `PUT /restaurant/menu/categories/:id` — update/reorder
- [ ] `DELETE /restaurant/menu/categories/:id`
- [ ] `POST /restaurant/menu/items` — create item
- [ ] `PUT /restaurant/menu/items/:id` — update (name, price, photo, availability)
- [ ] `DELETE /restaurant/menu/items/:id`
- [ ] Photo upload: presigned R2 URL flow
- [ ] `PATCH /restaurant/profile` — operating hours, open/closed toggle
- [ ] Restaurant dashboard UI: menu editor screen

### 2.2 Customer app — browse + cart
- [ ] `GET /restaurants` — list by proximity (requires customer lat/lng)
- [ ] `GET /restaurants/:id/menu` — full menu with categories
- [ ] Customer app UI: home screen (restaurant list)
- [ ] Customer app UI: restaurant page (menu scroll)
- [ ] Customer app UI: item detail modal
- [ ] Customer app UI: cart screen (Zustand cart state)

### 2.3 Checkout + order creation
- [ ] `POST /orders` — full order creation flow:
  - [ ] Server-side delivery fee calculation (Google Distance Matrix)
  - [ ] Stripe PaymentIntent creation
  - [ ] Order record created with item snapshot
  - [ ] Payment record created
  - [ ] `transfer_group` set on PaymentIntent
- [ ] Customer app UI: checkout screen (address, tip, total)
- [ ] Customer app UI: Stripe card entry (Stripe React Native SDK)
- [ ] Customer app UI: order confirmation screen

### 2.4 Restaurant order queue
- [ ] Socket.io: restaurant joins room on login (`restaurant:${userId}`)
- [ ] `restaurant:new_order` event emitted on order creation
- [ ] `PATCH /orders/:id/status` — PENDING → RESTAURANT_ACCEPTED (restaurant role)
- [ ] 3-minute auto-cancel timer (BullMQ job) if no acceptance
- [ ] `PATCH /orders/:id/status` — RESTAURANT_ACCEPTED: set prep time, mark READY
- [ ] Restaurant dashboard UI: orders panel (incoming queue)
- [ ] Restaurant dashboard UI: order detail modal

### 2.5 Driver app — online + dispatch
- [ ] `POST /drivers/online` — toggle, start location broadcast
- [ ] `POST /drivers/offline` — toggle, stop broadcast
- [ ] WebSocket: driver sends GPS every 10s while online → `GEOADD drivers:online`
- [ ] `PATCH /drivers/pricing` — update baseFee, perMileFee, minimumPayout
- [ ] Dispatch BullMQ job: candidate query → score + rank → FCM push to top 5
- [ ] 45-second accept window with 3-round fallback (SPEC.md: Dispatch System)
- [ ] `PATCH /orders/:id/status` — DRIVER_ASSIGNED → PICKED_UP → DELIVERED (driver role)
- [ ] Driver app UI: dashboard (online toggle, status)
- [ ] Driver app UI: incoming order modal (45s countdown)
- [ ] Driver app UI: active order screen (map + status buttons)

### 2.6 Real-time tracking
- [ ] Socket.io order room: customer joins on order creation
- [ ] `driver:location_update` emitted to order room every 10s
- [ ] `order:status_changed` emitted on every status transition
- [ ] Customer app UI: order tracking screen (live map + status bar)

---

## Phase 3 — Payments & Subscriptions (Weeks 5–6)

### 3.1 Stripe Connect onboarding
- [ ] `POST /connect/driver/onboard` — create Express account + return onboarding URL
- [ ] `POST /connect/restaurant/onboard` — create Express/Standard account + return URL
- [ ] Webhook `account.updated` handler: sync verification status
- [ ] Driver app UI: Stripe Express onboarding (WebView)
- [ ] Restaurant dashboard UI: Stripe Connect setup flow

### 3.2 Subscriptions
- [ ] `POST /subscriptions/create` — create Stripe Subscription, store on DB
- [ ] `DELETE /subscriptions/cancel` — cancel at period end
- [ ] Subscription enforcement middleware (see SPEC.md: Subscription Enforcement)
- [ ] Webhook handlers: `subscription.created`, `subscription.past_due`, `subscription.deleted`
- [ ] 3-day grace period logic for `PAST_DUE`
- [ ] Driver app UI: subscription selection + payment screen
- [ ] Driver app UI: subscription management screen
- [ ] Restaurant dashboard UI: subscription selection + management

### 3.3 Payout on delivery
- [ ] On `DELIVERED` status transition: fire both Stripe Transfers (SPEC.md: On DELIVERED)
- [ ] Store `restaurantTransferId` and `driverTransferId` on payment record
- [ ] Set `payment.status = TRANSFERRED`
- [ ] Webhook `transfer.failed` handler: alert admin queue

### 3.4 Refunds
- [ ] Auto-refund on PENDING → CANCELED (before transfers)
- [ ] `POST /admin/orders/:id/refund` — partial or full refund
- [ ] Transfer reversal logic for post-delivery refunds
- [ ] All refund scenarios from SPEC.md: Refund matrix

### 3.5 Earnings + payout history
- [ ] `GET /drivers/earnings` — today / week / month totals
- [ ] `GET /drivers/payouts` — itemized by order
- [ ] `GET /restaurant/payouts` — payout history
- [ ] Driver app UI: earnings dashboard
- [ ] Driver app UI: payout history list
- [ ] Restaurant dashboard UI: payout history screen

### 3.6 Admin panel
- [ ] `GET /admin/users` — filterable by role/status
- [ ] `PATCH /admin/users/:id/status` — suspend/ban
- [ ] `GET /admin/orders` — full log
- [ ] `PATCH /admin/subscriptions/:id` — override status/plan
- [ ] `GET /admin/transfers/failed` — failed transfers queue
- [ ] Admin panel UI: all screens (dashboard, users, orders, payments)

---

## Phase 4 — QA & Launch (Weeks 7–8)

### 4.1 Ratings
- [ ] `POST /ratings` — customer submits post-delivery rating
- [ ] Rolling average update on `driverProfile.rating` and `restaurantProfile.rating`
- [ ] Customer app UI: rating screen (post-delivery prompt)
- [ ] Driver app UI: ratings summary screen

### 4.2 Edge cases
- [ ] Payment failure at checkout: error handling + retry
- [ ] Restaurant goes offline mid-session: cart invalidation
- [ ] Driver disconnects mid-order: 2-min reconnect window + reassign
- [ ] Item marked unavailable after order placed: customer notification + cancel/continue

### 4.3 Security
- [ ] Rate limiting on all auth routes (Fastify rate-limit plugin)
- [ ] Input validation on all routes (Zod schemas)
- [ ] Stripe webhook signature verification (already in 1.4 — confirm in prod)
- [ ] Stripe Radar rules enabled on platform account
- [ ] API response sanitization: never expose passwordHash, stripeCustomerId, stripeAccountId
- [ ] Admin panel: IP allowlist or VPN requirement

### 4.4 Launch
- [ ] Switch Stripe to live mode
- [ ] EAS Build: iOS + Android production builds
- [ ] App Store submission
- [ ] Google Play submission
- [ ] Restaurant dashboard: Vercel production deploy
- [ ] Admin panel: Vercel production deploy (restricted access)
- [ ] Sentry error monitoring connected to all apps
- [ ] Onboard pilot cohort: 10+ restaurants, 20+ drivers manually

---

## Notes / Decisions Log

> Add any implementation decisions, deviations from SPEC.md, or important context here.
> Format: [DATE] — [DECISION]

- [2026-04-20] — Phase 1.1 scaffold completed with Turborepo workspace, two Expo apps, two Vite apps, Fastify API package, shared package, and initial `/health` route.

---

## Known Issues / Blockers

> Track anything that needs to be resolved before moving forward.

- None yet
