# WhatsApp Order Confirmation — Implementation Plan (V1)

## 1. What we're building
A public Shopify App Store app. When a customer places an order, the app:

1. Puts the order's **fulfillment order(s) on hold** so nothing can ship.
2. Sends a WhatsApp **template message** (business-initiated) asking the customer: *"Your order #1041 was placed. Reply YES to confirm and ship, or NO to cancel."*
3. Holds the order until the customer replies.
4. **YES** → releases the hold (manual fulfillment by merchant/warehouse).
5. **NO** → cancels the order. **No reply within the window (default 24h)** → auto-cancels.

All state and decisions are surfaced to the merchant in a Polaris admin dashboard.

---

## 2. Research findings & key constraints

### WhatsApp (Meta Cloud API, chosen over BSP)
- **Direct Cloud API** (`graph.facebook.com/<version>/<phone-id>/messages`) is free to use; no SDK required (plain `fetch`). Meta bills per delivered template message by category + destination country (~$0.02–0.05 USD typical). BSPs (Twilio/360dialog) add ~$0.005/msg on top → skipped.
- **Templates are the only way to message outside the 24h customer-service window.** The order-confirmation message must be an **approved template** (category `utility`). Approve it once in WhatsApp Manager, then send by name + params. Body can include `{{1}}` order number placeholder.
- **Interactive reply buttons** on the template (payloads like `CONFIRM_YES` / `CONFIRM_NO`, carrying a per-order token) give clean reply matching vs. free-text "yes"/"no". Support both: buttons + free text.
- **Inbound messages** arrive via a webhook we must expose publicly (HMAC-signed with `X-Hub-Signature-256`, plus a one-time challenge/verify token).
- **Phone number hosting:** the business needs a WABA + a phone number. Dev/test number from the Meta app dashboard works during development. Production needs the merchant's own WABA, a verified Meta Business, and either a virtual number (Twilio/Telnyx) or an existing non-WhatsApp number (migration).
- **Pricing notes (2026):** incoming messages free; in-window service/utility replies free only until Sep 30, 2026 then ~$0.02–0.05/msg; 1,000 free service messages/month per number from Oct 2026; template sends outside window always billed. Order-confirmation utility sends are the primary cost driver → put under Shopify billing.

### Shopify
- **Fulfillment holds are the exact primitive we need.** `fulfillmentOrderHold` (GraphQL) blocks the order from being fulfilled at the platform level; `fulfillmentOrderReleaseHold` unblocks it. This is how existing hold/preorder apps (SweetRelease, Addora, Dualflow) work.
- **Do NOT act on `orders/create` alone** — fulfillment orders are only created after order routing. Best practice: subscribe to **`orders/create`** (record + create hold best-effort) **and** **`fulfillment_orders/order_routing_complete`** (idempotently hold + send WhatsApp when a fulfillment order is ready).
- **Cancel path:** `orderCancel` (needs `write_orders`); canceling a *paid* order also needs an `orderRefund`/capture-void — confirm behavior during implementation. Recommend `CANCEL_INVENTORY` + refund for paid, `CANCEL` for unpaid.
- **Customer phone** is `order.phone` (captured in checkout; often missing country code). **This is the WhatsApp number match key.** Must normalize (default country from order/merchant setting) and compare against inbound `wa_id` (E.164 with country code).
- **Public-app constraints:** scopes must be justified in App Store review; holds must use `notifyMerchant: true`; app cannot modify checkout (so we rely on the merchant requiring the checkout phone field). Precedent apps confirm this hold pattern passes review.

---

## 3. Tech stack (mostly already in the repo)
- **Kept:** React Router (Remix fork) + Polaris + App Bridge, `@shopify/shopify-app-react-router` (auth/webhooks/sessions), Prisma.
- **DB:** migrate `prisma/schema.prisma` datasource from SQLite → **PostgreSQL** (SQLite is dev-only; a public multi-tenant app needs a hosted PG — Fly.io/Railway/Supabase).
- **Hosting:** persistent process host (Fly.io/Railway) so an in-process scheduler can run — serverless (Vercel) would need an external cron instead (design supports both via a protected cron endpoint).
- **New deps:** none strictly required (WhatsApp via `fetch`). Optionally `pino` + `pino-http` for structured webhook logging.
- **Secrets:** `SHOPIFY_API_KEY/SECRET/SCOPES/APP_URL`, `DATABASE_URL`, plus per-shop WhatsApp credentials (below), stored **encrypted** (fernet via `@fernet/fernet`-style key in env).

---

## 4. Data model (Prisma)
```prisma
enum ConfirmationStatus { PENDING CONFIRMED CANCELLED_BY_CUSTOMER CANCELLED_TIMEOUT CANCELLED_MANUAL DELIVERY_FAILED }

model AppSettings {          // one row per shop
  id          String  @id      // shop domain
  enabled     Boolean @default(true)
  defaultCountryCode String @default("US")
  defaultPhoneMinLength Int @default(7)
  templateName    String @default("order_confirmation")
  templateLang    String @default("en_US")
  expiryHours     Int @default(24)
  remindAtHours   Int?         // optional reminder before expiry
  autoCancelOnTimeout Boolean @default(true)
  waPhoneNumberId String?      // merchant's WABA phone number id
  waAccessToken   String?      // encrypted system-user token
}

model OrderConfirmation {
  id          String  @id @default(cuid())
  shop        String
  orderId     String            // gid://shopify/Order/…
  orderName   String            // #1041
  customerPhone String          // normalized, E.164
  whatsappWaId String?          // wa_id of whoever replied
  status      ConfirmationStatus @default(PENDING)
  templateMsgId String?
  holdHandles String?           // JSON array of handle(s) placed
  expiresAt   DateTime
  confirmedAt DateTime?
  cancelReason String?
  payload     String?           // raw webhook json (debug)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  @@unique([shop, orderId])
  @@index([status, expiresAt])
}
```

---

## 5. Backend implementation

### 5.1 Shopify webhooks (add to `shopify.app.toml` + routes)
- `orders/create` → `app/routes/webhooks.orders.create.tsx`: upsert `OrderConfirmation` (PENDING), fire-and-forget hold-if-possible.
- `fulfillment_orders/order_routing_complete` → `webhooks.fulfillment_orders.order_routing_complete.tsx`: idempotent **hold** (create if no open hold) + **send WhatsApp template**. This is the reliable trigger.
- Keep existing `app/uninstalled` cleanup; add cleanup of confirmations/settings rows.

### 5.2 Services (`app/services/`)
- **`shopify/confirmations.ts`** — upsert record, place hold (list open fulfillment orders via `fulfillmentOrders` query with `query: order_id:<id>`, then `fulfillmentOrderHold` per order with `reason: CUSTOMER_CONFIRMATION_REQUESTED` (fallback `OTHER`), `reasonNotes: "Awaiting WhatsApp confirmation"`, `notifyMerchant: true`, unique `handle: wa-confirm-<orderId>`), release (`fulfillmentOrderReleaseHold`), confirm/cancel orchestration. Verifies `status == OPEN` + no active `fulfillmentHolds` before fulfillment (per Shopify guidance).
- **`whatsapp/send.ts`** — POST to `/{phone-id}/messages`: `type: template`, name/lang, body param = order number, interactive reply buttons `CONFIRM_YES|<orderid-token>` / `CONFIRM_NO|…`. Records `templateMsgId` (for status tracking). Cyrillic/unicode-safe.
- **`whatsapp/webhook.ts`** —
  - `GET` verify: check `hub.verify_token` against env, return `hub.challenge`.
  - `POST`: verify `X-Hub-Signature-256` with `META_WA_APP_SECRET`; parse inbound `messages[]` (text or `interactive.button_reply`); extract `from` (wa_id), strip `wa_` prefix, normalize.
- **`confirmation/reply.ts`** — match reply → pending `OrderConfirmation`:
  1. button payload token → exact order.
  2. free-text: parse YES/NO words (i18n list, trim, case-insensitive); if text contains an order number, match that, else most recent PENDING for that wa_id/phone.
  3. YES → status CONFIRMED → **release hold** (manual fulfillment).
  4. NO → status CANCELLED_BY_CUSTOMER → **cancel + refund/void**.
  - Send a short **free-form reply** within the 24h window acknowledging ("Your order will ship…"/"Order cancelled."). Reply is free until the Oct 2026 change; cheap after.
- **`confirmation/expiry.ts`** — select PENDING & `expiresAt < now()`: auto-cancel (default) or notify merchant. Also handle `DELIVERY_FAILED` from message-status webhooks (WhatsApp `messages` field, `status: failed`) → notify merchant, leave hold, store reason.
- **`scheduler.ts`** — in-process `setInterval` (every 5 min) + protected `/internal/scheduler` route (env bearer token) for external crons; both call `expiry.ts`. Optional reminder send before expiry.

### 5.3 Scheduler/status webhook (WhatsApp field `messages` + `message status` on WABA app)
- Sample incoming-message shape (button reply):
  ```json
  { "object":"whatsapp_business_account","entry":[{"changes":[{"field":"messages","value":{"metadata":{"display_phone_number":"…","phone_number_id":"…"},"messages":[{"from":"15551234567","type":"interactive","interactive":{"type":"button_reply","button_reply":{"id":"CONFIRM_YES|<orderid-token>","title":"Yes"}}}],"messages":[{"status":"delivered","id":"wamid…"}]}}]}]}
  ```

---

## 6. Admin UI (Polaris, under existing `/app` routes)
- **Settings page** (`/app/settings/route.tsx`): enable toggle, default country, template name/lang, expiry hours, reminder toggle, auto-cancel-on-timeout toggle, WhatsApp number id + access token fields (encrypted at rest), connection status/"Test message" button.
- **Dashboard page** (`/app/confirmations/route.tsx`): Polaris `IndexTable` of pending/confirmed/cancelled confirmations with order name, customer phone, status badge, timers; filters + bulk actions (release hold, cancel, mark manual).
- **Onboarding flow**: pre-install required; on first `authenticate`, guide merchant through WABA setup + template approval checklist.

---

## 7. Billing (required for public apps)
- Recurring monthly subscription via `appSubscriptionCreate` (e.g. $9–15/mo). Gate app features on active subscription using `shopify.session.offlineAccessAccount`/subscription check helper; `app/subscription` webhook for cancellations. Define plan in Partner dashboard first.

---

## 8. WhatsApp & Meta setup (manual prerequisites, checklist for the merchant)
1. Meta Developer account → create app → connect a **Meta Business** + **WhatsApp Business Account (WABA)**.
2. Get a **phone number**: dev/test number (Meta dashboard) or production number (Twilio/Telnyx virtual, or migrate existing).
3. Create **`order_confirmation` template** (category `utility`, language `en_US`, body with `{{1}}` order-number placeholder, two interactive reply buttons) → submit for approval.
4. Configure **webhook**: endpoint `https://<app-url>/api/whatsapp/webhook`, subscribe to `messages` subfield + message status, set verify token + app secret.
5. Get **system-user** access token (long-lived) + `phone_number_id` from Meta dashboard → enter in app Settings.
6. Merchant enables **"Phone (required)"** at Shopify checkout so `order.phone` is always present.

---

## 9. App Store submission checklist
- Privacy policy + app permissions (explain WhatsApp data + Shopify scope justification).
- Test/uninstall lifecycle, webhook idempotency, hold stays if app down, re-send/retry logic, logging (no PII in logs).
- Compliance: Meta opt-in requirement — order confirmation triggers are the standard utility case; surface a consent line in onboarding; customer can reply to opt out.

---

## 10. Milestones
1. **M0 Meta/Shopify sandbox:** WABA + dev number + approved template; wire env vars; scaffolding routes.
2. **M1 Core hold flow:** orders webhooks + hold + template send + delivery-fail handling (with test orders in dev store).
3. **M2 Reply handling:** Meta webhook verify + inbound + confirm/cancel orchestration + manual fulfillment release.
4. **M3 Admin UI + settings + onboarding + billing.**
5. **M4 Expiry/reminders via scheduler; delivery-status tracking; edge cases (no phone, non-WhatsApp number, number changed, multiple orders).**
6. **M5 Postgres + production deploy + App Store submission.**

## 11. Open decisions to confirm during build
- Exact `fulfillmentOrderHold` hold `reason`/duration limits for **public** apps (verify against current API; may need `OTHER` + notifyMerchant).
- Paid-order cancel: refund via `orderRefund` vs. capture-void — decide per payment-window behavior.
- Self-serve per-merchant WABA credentials vs. app-owner-operated WABA (single shared sender number) — per-merchant recommended for multi-tenant scale.