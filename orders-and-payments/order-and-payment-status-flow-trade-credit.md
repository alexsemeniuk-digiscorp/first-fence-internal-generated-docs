# Order & Payment Status Flow — with focus on Trade (Credit) Accounts

How an order's **status** and **payment** fields are set and later changed, end to end.
The credit-account path is traced line by line; the other payment methods are included
as a comparison table because they share the same columns and the same email/notification
machinery.

Companion doc: [`credit-account-flow.md`](./credit-account-flow.md) covers the read-only
`GET /api/customer-portal/customers/credit-account` balance endpoint. This doc covers the
**write** side — what actually mutates order state when a trade customer pays on credit.

---

## 1. The data model — three columns, two lookup tables

Status and payment are **not** one field. An order carries three separate integers, and
different parts of the system read different ones. Getting this wrong is the single
biggest source of confusion in this area.

| Column | Type | FK? | Read by |
|--------|------|-----|---------|
| `orders.status` | int | ✅ → `order_statuses.id` | Admin UI, `orderStatus` relation, confirmation-email subject line |
| `orders.selected_payment` | int | ❌ **no FK** — bare int | The **email template branching** (`order.edge`) |
| `orders.payment` | int | ✅ → `payments.id` | `selectedPayment` relation, admin display |

Schema: `database/migrations/1571924481804_order_schema.js`
(`status` at :55, `selected_payment` at :12, `payment` at :74).

### 1.1 `order_statuses` — the status lookup

Seeded in `database/seeds/OrderStatusSeeder.js:18`. Note the mismatch: the table has
**13** rows, but `app/Utils/constants.js` only names **10** of them.

| id | `status` | Constant in `app/Utils/constants.js` |
|----|----------|--------------------------------------|
| 1 | `completed` | `STATUS_COMPLETED` |
| 2 | `pending` | `STATUS_PENDING` |
| 3 | `failed` | `STATUS_FAILED` |
| 4 | `cancelled` | `STATUS_CANCELED` *(note the spelling difference: constant is `CANCELED`, row is `cancelled`)* |
| 5 | `test` | `STATUS_TEST` |
| 6 | `closed` | `STATUS_CLOSED` |
| 7 | `sales` | `STATUS_SALES` |
| 8 | `created` | `STATUS_CREATED` |
| 9 | `requested` | `STATUS_REQUESTED` |
| 10 | `spam` | `STATUS_SPAM` |
| 11 | `refunded` | — **no constant** |
| 12 | `duplicate` | — **no constant** |
| 13 | `lead` | — **no constant** |

Statuses 11–13 are reachable **only** through the admin status endpoint (which accepts any
`statusID` that exists in the table). No server-side code path sets them.

### 1.2 The two payment enumerations

`app/Utils/constants.js:13-24` defines **two parallel sets** for the same five methods —
the legacy bare int and the FK value. They are off by one.

| Method | `selected_payment` (legacy, no FK) | `payment` (FK → `payments.id`) | `payments.value` |
|--------|-----------------------------------|-------------------------------|------------------|
| Card | `PAYMENT_CARD` = **0** | `PAYMENT_CARD_NEW` = **1** | `CARD` |
| Bank transfer | `PAYMENT_BANK` = **1** | `PAYMENT_BANK_NEW` = **2** | `BANK` |
| PayPal | `PAYMENT_PAYPAL` = **2** | `PAYMENT_PAYPAL_NEW` = **3** | `PAYPAL` |
| iwoca | `PAYMENT_IWOCA` = **3** | `PAYMENT_IWOCA_NEW` = **4** | `IWOCA` |
| **Trade credit** | `PAYMENT_CREDIT` = **4** | `PAYMENT_CREDIT_NEW` = **5** | `CREDIT` |

`payments` rows are seeded in `database/seeds/PaymentStatusSeeder.js:18`.

**Both columns must always be written together.** The confirmation email branches on the
*legacy* column, not the FK:

```
resources/views/emails/order/order.edge:81-90
  @if(order.selected_payment == 1)        → bank-payment component
  @elseif(order.selected_payment == 4)    → credit-payment component   ← trade credit
  @else                                   → progress-bar + other-payments
```

So if a future refactor writes only `order.payment = PAYMENT_CREDIT_NEW` and forgets
`selected_payment = PAYMENT_CREDIT`, the credit customer silently receives the generic
card-style receipt with a payment progress bar instead of the "placed on your credit
account" wording.

---

## 2. Lifecycle overview

```
                        ┌──────────────────────────────────────────┐
                        │  cart (carts.completed = false)          │
                        └───────────────────┬──────────────────────┘
                                            │ POST /api/order/create
                                            │   (OrderController.save)
                                            ▼
                              status = 8  created         ← STATUS_CREATED (default)
                              selected_payment = NULL
                              payment = NULL
                                            │
        ┌───────────────────┬───────────────┼───────────────┬────────────────────┐
        │ card / wallet     │ bank transfer │ trade CREDIT  │ paypal             │ iwoca
        ▼                   ▼               ▼               ▼                    ▼
  1 completed          2 pending       1 completed     1 completed / 4    1/3/2 per
  or 3 failed         (awaiting        (immediately)    cancelled          iwoca state
                       funds)
        │                   │               │               │                    │
        └───────────────────┴───────────────┴───────┬───────┴────────────────────┘
                                                    │
                                                    ▼
                            Admin may override to ANY order_statuses row
                            (PUT /api/admin/order/status/:id)
                            incl. refunded (11), duplicate (12), lead (13)
```

Key point for credit: **there is no intermediate state.** A trade credit order goes
`created → completed` in a single request. There is no "awaiting approval", no pending
state, and no asynchronous callback — unlike bank transfer (`pending`), card (3DS
round-trip) or iwoca (polled third-party state).

---

## 3. Stage 1 — order creation (shared by all payment methods)

**`POST /api/order/create`** → `OrderController.save`
(`app/Controllers/Http/OrderController.js:487`), middleware `verifyCart`.

1. `verifyCart` (`app/Middleware/VerifyCart.js:20`) resolves the cart identity (cookie
   session **or** the mobile cart-token header), loads the single `carts` row where
   `sessionID = identity AND completed = false`, and injects it as `request.body.cart`.
   No cart → error response (404 for token callers, 200-with-`errors` for cookie callers).
2. Creates `deliveries`, `addresses` ×2, `people` rows.
3. `total_price` = `getProductsTotalPrice(productsJson)` — **products only, ex-VAT**.
4. Creates the `orders` row with `status = STATUS_CREATED` (8) — overridable by a
   `status` field in the request body (`OrderController.js:501`).
   `selected_payment` and `payment` are left **NULL**.
5. Fires notification `order.created` and Salesforce `salesforce::createWebActivity`.
6. Creates the `order_details` row.

The quote-conversion variant does the same thing at
`app/Controllers/Http/BasketQuoteController.js:1795-1814` — also `status: STATUS_CREATED`,
plus `basket_quote_id` set (which is what later authorises the `/quote-pay/*` routes).

At this point the order exists, is unpaid, and its cart is still open.

---

## 4. Stage 2 — the trade credit payment flow (step by step)

**Routes** (both land on the same controller method):

| Route | Middleware | Source |
|-------|-----------|--------|
| `POST /api/orderCreditPayment` | `auth`, `verifyCart` | `start/routes/public.js:49` |
| `POST /api/quote-pay/orderCreditPayment` | `auth`, `verifyQuoteCart` | `start/routes/public.js:139-142` |

Handler: `CreditPaymentController.process`
(`app/Controllers/Http/CreditPaymentController.js:104`).

### Step 0 — middleware

- **`auth`** = `App/Middleware/ApiTokenAuth` (registered in `start/kernel.js:56`). It only
  checks `ctx.auth.user` is populated and 401s otherwise. The scheme is the `api` personal-token
  authenticator (`config/auth.js:21`). Credit is the **only** payment route that requires
  authentication — bank/paypal/card are all guest-capable.
- **`verifyCart`** injects the open cart (as in Stage 1), **or**
- **`verifyQuoteCart`** (`app/Middleware/VerifyQuoteCart.js:6`) takes `ORDER_ID` from the
  body, loads the order, **rejects it unless `order.basket_quote_id` is set**, then injects
  `Cart.find(order.cart_ID)` — notably *without* the `completed = false` filter that
  `verifyCart` applies.

### Step 1 — PO number gate (`:107-111`)

```js
if (!po_number || !po_number.trim()) → 400 { errors: [{ message: "PO number is required." }] }
```

Trade credit is the **only** flow with a mandatory PO number. This is a hard 400 — the one
genuine HTTP error status in the method; every later failure returns **200** with an
`errors` array.

### Step 2 — trade account resolution (`:113-120`)

```js
const user = await auth.getUser();
const cardCode = user.CardCode;          // users.CardCode → SAP business partner code
if (!cardCode) → { errors: [{ message: "No trade account found." }] }
```

`users.CardCode` (`database/migrations/1503248427885_user.js:15`) is the link to SAP. A
logged-in retail customer with no `CardCode` is stopped here.

### Step 3 — SAP credit profile fetch (`:122-128`)

```js
const profile = await SAPService.getProfile(cardCode);
if (!profile) → { errors: [{ message: "Unable to retrieve credit account details." }] }
```

`SAPService.getProfile` (`app/Services/SAPService.js:4-8`) runs
`SELECT * FROM "OCRD" WHERE "CardCode" = ? LIMIT 1` against SAP HANA.

> ⚠️ **This guard is dead code.** `getProfile` returns `{}` — an empty *object* — when no
> row matches, and `{}` is truthy. The `!profile` branch can never fire. A missing SAP
> business partner therefore falls through to Step 4 and surfaces as
> *"No credit account available."* instead of *"Unable to retrieve credit account details."*

### Step 4 — credit limit check (`:130-136`)

```js
const creditLimit = parseFloat(profile.CreditLine);
if (!creditLimit) → { errors: [{ message: "No credit account available." }] }
```

Catches `NaN` (missing SAP row, per the note above), `0`, and empty. This is the real
"do you have a credit account at all" gate.

### Step 5 — available balance (`:138-142`)

```
availableBalance = CreditLine − (Balance + DNotesBal + OrdersBal)
```

Identical formula to the read-only balance endpoint documented in `credit-account-flow.md`.
All four values come live from SAP `OCRD` on every request — nothing is cached locally.

> ⚠️ If any of `Balance` / `DNotesBal` / `OrdersBal` is absent from the SAP row,
> `parseFloat` yields `NaN`, `availableBalance` becomes `NaN`, and the Step 8 comparison
> `NaN < totalWithVAT` evaluates to **false** — i.e. the insufficient-credit check is
> skipped and the order completes. There is no `isNaN` guard on the balance components
> (there is one on `hireDeposit` / `installationPrice`).

### Step 6 — order load (`:144-162`)

Loads the order by `ORDER_ID` with `person`, both addresses, `cart`, `delivery` +
products/options/installation surcharges/offloads. Missing order → `Grafana.emerg` +
`{ errors: [{ message: "Invalid Order ID!" }] }`.

> ⚠️ **No ownership check and no state check.** The order is fetched by raw `ORDER_ID`
> with no `where user_id = user.id`, and its current `status` is never inspected. Nothing
> prevents (a) an authenticated trade user from completing *another* customer's order, or
> (b) the same order being re-posted after it is already `completed` — each replay re-runs
> the mutation, re-fires the confirmation email and re-notifies admins. On the
> `verifyCart` route the replay is partly limited by the cart having been flipped to
> `completed = true` in Step 10 (the middleware then 404s), but the `/quote-pay/` route's
> `verifyQuoteCart` does **not** filter on `completed`, so it stays replayable.

### Step 7 — total recalculation (`:164-182`)

Prices are **recomputed server-side** from the cart; nothing money-related is taken from
the request body.

```js
orderTotalPrice    = getProductsTotalPrice(cart products)        // ex-VAT, products only
deliveryPriceTotal = getDeliveryPriceTotal(orderJson)            // delivery + offload
totalForComparison = orderTotalPrice + deliveryPriceTotal
                   + (hireDeposit / 1.2)        if set & numeric  // de-VAT'd: deposit is VAT-inclusive
                   + installationPrice          if set & numeric  // NB: installationRemovalPrice is NOT added
totalWithVAT       = totalForComparison * 1.2                     // VAT = 1.2
```

Two quirks worth knowing:

- `installationRemovalPrice` is **excluded** from the credit comparison, even though
  `computeOrderChargeTotal` (`app/Utils/price-calculations.js:99-107`) and `mapOrder`
  (`app/Utils/order-helpers.js:110-118`) both include it in the amount actually invoiced.
  The credit check therefore under-counts orders with installation removal.
- The hire deposit is divided by VAT then the whole subtotal is multiplied by VAT, so the
  deposit is effectively taxed here — whereas `computeOrderChargeTotal:96` deliberately
  *excludes* the deposit from tax (`excludedTax`). The two totals can differ.

### Step 8 — the credit decision (`:184-192`)

```js
if (availableBalance < totalWithVAT)
  → 200 { errors: [{ message: "Insufficient credit balance to complete this order." }] }
```

No partial payment, no fallback to another method, no admin-approval path. Order status is
left untouched at `created` (8), so the customer can pick a different payment method.

### Step 9 — the status mutation (`:194-199`)

This is the single write that moves the order:

```js
order.status           = STATUS_COMPLETED;     // 1
order.selected_payment = PAYMENT_CREDIT;       // 4  → drives the email template
order.payment          = PAYMENT_CREDIT_NEW;   // 5  → FK to payments row "CREDIT"
order.po_number        = po_number.trim();     // trade-credit only
order.total_price      = orderTotalPrice;      // ← re-stamped, products only, ex-VAT
await order.save();
```

Note `total_price` is overwritten with the **products-only ex-VAT** figure — delivery,
installation and hire deposit are *not* folded in. Downstream totals
(`grandTotal` in `mapOrder`, `computeOrderChargeTotal`) rebuild the full charge from
`total_price` + the delivery/installation/deposit columns, so this is consistent with the
rest of the codebase — but `orders.total_price` alone is **not** the amount charged.

There is no DB transaction around this write and the cart update that follows.

### Step 10 — close the cart (`:201-226`)

Resolves `cart_ID` (falling back to the loaded relation), loads the `Cart`, sets
`completed = true`, saves. Wrapped in `try/catch` — a failure logs to Grafana
(`error` / `warn`) but **does not** roll back the completed order. A missed cart update
leaves the customer's basket live against a paid order.

> Stylistic note: every other controller does this in one line —
> `await order.cart().update({ completed: true })` (e.g. `IwocaController.js:192`,
> `orderFinalisation.js:51`). Credit is the only one with the defensive load-and-save block.

### Step 11 — relation hydration (`:228-234`)

Lazily loads `orderStatus` and `selectedPayment` if not already loaded, so the JSON
returned to the client and passed to the email carries the human-readable labels
(`orderStatus.status = "completed"`, `selectedPayment.value = "CREDIT"`).

### Step 12 — admin notification (`:239-253`)

```js
NotificationService.notify({
  typeKey: "order.payment_selected",
  title:  `Credit Payment Placed for Order #<id>`,
  body:   `Order #<id> has been placed on credit account. PO: <po_number>`,
  data:   { order_id, payment_method: "credit", total_amount,
            previous_status: "created", new_status: "completed" },
  url:    `/orders/<id>`,
})
```

`NotificationService.notify` (`app/Services/NotificationService.js:11`) is **fire-and-forget**
via `setImmediate` — it never blocks or fails the payment. It fans out to **every** user in
`users` across every channel in `notification_channels` (IN_APP / EMAIL / PUSH), honouring
`notification_preferences` and falling back to the type's `default_enabled`. In-app
notifications are also broadcast over the `notifications:<userId>` websocket topic.

> Credit uses `order.payment_selected`. Card/PayPal/Worldpay use `order.completed`
> (`WorldpayController.js:110,274`, `CardPaymentController.js:144`,
> `PayPalController.js:365`). So an admin who has muted `order.payment_selected` will not
> be told about a completed credit order, and dashboards filtering on `order.completed`
> will under-count credit sales. Both keys **are** seeded
> (`database/seeds/NotificationTypeSeeder.js:19,34`).
>
> Separately — not a credit concern but part of the same status machinery — the
> `order.failed` key used by `CardPaymentController.js:199` and `WorldpayController.js:339`
> is **not** seeded. `_sendNotification` looks the type up by key and `return`s early when
> it finds nothing (`NotificationService.js:25-26`), so **failed-payment notifications are
> silently dropped**. `order.cancelled` and the whole `payment.*` family are commented out
> in the seeder too.

### Step 13 — customer confirmation email (`:255`)

```js
Event.fire("send::order", finalOrderJson);
```

Wired at `start/events.js:3` → `Order.send` (`app/Listeners/Order.js:10`):

- Runs `mapOrder` to build display totals (`grandTotal = totalPrice * 1.2 − excludedTax`,
  `app/Utils/order-helpers.js:121`).
- Renders `emails.order.order`, which branches on `selected_payment == 4` and pulls in
  `resources/views/emails/order/credit-payment.edge` — "This order has been placed on your
  credit account", the PO reference (`credit-payment.edge:21-25`), and Total Payable inc VAT.
  Deliberately **no** payment progress bar.
- Subject: `First Fence Ltd - Order Confirmation #<id>`
  (it becomes `Payment Declined Order #<id>` when `orderStatus.status === "failed"`,
  `Order.js:24-30` — unreachable for credit, which never sets `failed`).
- To: customer, cc `weborders@` + `salesadmin@`.
- Any throw is swallowed and logged via `Grafana.emerg`.

### Step 14 — auto-assign an account manager (`:257-260`, `:268-313`)

Non-blocking (`.catch(() => {})`), unique to the credit flow:

1. Get a Salesforce token (`SalesforceTokenService`).
2. `SELECT Id, Name, SAP_BP_Code__c, Primary_Sales_Person__r.Username FROM Account
   WHERE SAP_BP_Code__c = '<cardCode>'`.
3. Match `Primary_Sales_Person__r.Username` (lower-cased, trimmed) against `users.email`.
4. `UPDATE orders SET assign_to = <adminUser.id>` — note this is a **query-builder update**,
   so no model hooks fire and **no `orders_changelog` entry is written** (contrast with the
   manual admin assignment path, which does log — `AssignUserToOrderController.js:88-93`).
5. `Event.fire("send::order-assigned", { orderId, poNumber, adminEmail })` →
   `start/events.js:7` → `Order.sendAssignment` (`app/Listeners/Order.js:194`) → renders
   `emails.orderAssigned` with subject `Credit Order Assigned - Order #<id>`, showing the
   PO number (`resources/views/emails/orderAssigned.edge:12-13`).

Every step is inside `try/catch` → `Grafana.error`. Salesforce being down never fails a
credit order.

### Step 15 — response (`:262-265`)

```
{ "message": "Order has been successful!", "order": { <the saved order model> } }
```

---

## 5. Cross-method comparison

| Method | Controller | Terminal status(es) | `selected_payment` / `payment` | Cart closed | Auth required | PO required |
|--------|-----------|---------------------|-------------------------------|-------------|---------------|-------------|
| **Trade credit** | `CreditPaymentController:194-199` | `completed` (1) only | 4 / 5 | ✅ (guarded block) | ✅ `auth` | ✅ |
| Card (Realcontrol) | `CardPaymentController:101,177` | `completed` (1) / `failed` (3) | 0 / 1 | ✅ | ❌ | ❌ |
| Card (Worldpay HPP + native) | `WorldpayController:242,320` | `completed` (1) / `failed` (3) | 0 / 1 | ✅ | ❌ | ❌ |
| Bank transfer | `BankPaymentController:94-96` | **`pending` (2)** | 1 / 2 | ✅ | ❌ | ❌ |
| PayPal | `PayPalController:108,332,448` | `created` (8) → `completed` (1) / `cancelled` (4) | 2 / 3 | ✅ | ❌ | ❌ |
| iwoca | `IwocaController:187-196` | `completed` (1) / `failed` (3) / `pending` (2) | 3 / 4 | ✅ on completed only | ❌ | ❌ |

**Credit is the only method that is authenticated, PO-gated, SAP-gated, has a single
terminal state, and auto-assigns a salesperson.**

### 5.1 The shared finalisation helper (`app/Utils/orderFinalisation.js`)

New on the current branch (`feature/native-payment-routes`) for the native mobile payment
work. `finalisePaidOrder` (`:34`) centralises exactly the Stage-2 tail:

```js
order.selected_payment = selectedPayment;
order.payment          = payment;
order.status           = status;              // default STATUS_COMPLETED
await order.save();
if (markCartCompleted) await order.cart().update({ completed: true });
// then: load orderStatus, optional notification, Event.fire("send::order", orderJson)
```

`failPaidOrder` (`:93`) is the mirror image for `STATUS_FAILED`. Its header comment states
the intent to fold the existing Worldpay/Card/PayPal/iwoca/Bank/**Credit** controllers into
it. **`CreditPaymentController` does not use it today** — it still hand-rolls the same
sequence. When credit is migrated, the credit-specific extras that the helper does *not*
cover must be preserved: the `po_number` write, the `total_price` re-stamp, the
`order.payment_selected` notification key (the helper takes an arbitrary payload, so this
is fine), and the `_tryAutoAssignUser` call.

---

## 6. Stage 3 — post-checkout status changes (admin)

### 6.1 The canonical endpoint

**`PUT /api/admin/order/status/:id`** → `OrderStatusController.update`
(`app/Controllers/Http/OrderStatusController.js:37`), route `start/routes/admin.js:6`,
group middleware `["auth", "admin"]` with prefix `api/admin`.

Body: `{ statusID }`.

1. Validates `statusID` as `number|exists:order_statuses,id` — unless it is `0`, which is
   the sentinel for "clear the status" (`:43-53`).
2. Captures `priorStatusID` / `priorStatusName` **before** mutating, and computes
   `statusChanged` (`:62-69`).
3. Applies: `statusID === 0` → `order.orderStatus().dissociate()`, else
   `order.status = statusID`; then `order.save()` (`:72-78`).
4. **If and only if the status actually changed** (`:85-93`), writes an `orders_changelog`
   row (`database/migrations/1697529756485_orders_changelog_schema.js`) with
   `action = 'Status changed from "<prior>" to "<new>"'`, `user_id` = the acting admin,
   `assigned_to` = the order's current assignee. This is the order's Edit History.
5. Notifies `order.status_updated` to all subscribed admins (`:96-104`).
6. Returns the new status object. Any throw is caught and returned as
   `{ errors: [{ message: "Error updating order." }] }` — **200, not 5xx**.

This endpoint accepts **any** `order_statuses` id, which is how `refunded` (11),
`duplicate` (12) and `lead` (13) are reachable despite having no constants.

Because it only writes `orders.status`, a status change never touches
`selected_payment` / `payment` — a refunded credit order still reads as `CREDIT`, which is
correct, and the email template branching is unaffected.

### 6.2 The deprecated endpoint

`OrderController.update` (`app/Controllers/Http/OrderController.js:824`) is marked
`// @Deprecated`. It can set both `status` and `userID` (assignee) with the same
`0`-means-dissociate convention, but writes **no changelog** and sends **no notification**.
Prefer `OrderStatusController.update` + `AssignUserToOrderController.update`.

### 6.3 Assignment (adjacent, not a status change)

`PUT /api/admin/order/assignto/:id` → `AssignUserToOrderController.update`. Uses optimistic
concurrency — the caller must send `previousUserID` matching the stored assignee or it 400s
("User does not match previous user. Please refresh the page and try again."). Writes an
`orders_changelog` row, notifies `order.assigned`, and syncs `Assigned_Staff` to the
Salesforce Web_Activity (fire-and-forget).

The credit flow's Step 14 auto-assignment bypasses all of this — no changelog row, no
`order.assigned` notification, no optimistic check. It sends its own
`Credit Order Assigned` email instead.

### 6.4 SAP order number is manual

There is **no** automatic push of a completed order into SAP as a sales order.
`sap_order_numbers` rows are created only via `POST /api/admin/sap`
(`SapOrderNumberController.store`), i.e. by hand from the admin UI.

Consequence for credit accounts: completing a web credit order does **not** move SAP's
`OrdersBal`. Until someone books the order in SAP, the next credit check for that customer
still sees the pre-order balance — so a trade customer can place several orders in quick
succession that individually pass the check but collectively exceed their credit limit.
The only backstop is `parseFloat(profile.CreditLine)` being re-read live per request.

---

## 7. Gotchas summary (credit-specific)

| # | Where | Issue |
|---|-------|-------|
| 1 | `CreditPaymentController.js:124` | `if (!profile)` is unreachable — `SAPService.getProfile` returns `{}`, not `null`, on a miss. Misses surface as "No credit account available." |
| 2 | `CreditPaymentController.js:138-142`, `:184` | No `isNaN` guard on `Balance`/`DNotesBal`/`OrdersBal`; a `NaN` `availableBalance` makes `NaN < totalWithVAT` false, **skipping** the insufficient-credit check |
| 3 | `CreditPaymentController.js:144-162` | Order fetched by raw `ORDER_ID` — no `user_id` ownership check, no current-status check ⇒ cross-customer completion and replay are both possible; `/quote-pay/` is the more exposed route since `VerifyQuoteCart` omits the `completed = false` filter |
| 4 | `CreditPaymentController.js:178-180` | `installationRemovalPrice` omitted from the credit comparison though it *is* invoiced |
| 5 | `CreditPaymentController.js:174-182` | Hire deposit is effectively VAT'd in the credit comparison but VAT-exempted in `computeOrderChargeTotal` |
| 6 | `CreditPaymentController.js:194-226` | Status write and cart close are not in a transaction; a cart-update failure leaves a completed order with a live basket |
| 7 | `CreditPaymentController.js:240` | Uses `order.payment_selected`, not `order.completed` — credit sales are invisible to anything keyed on `order.completed` |
| 8 | `CreditPaymentController.js:297-299` | Auto-assign uses a query-builder `update`, so no `orders_changelog` entry is written for the assignment |
| 9 | Most failure branches | Return **HTTP 200** with an `errors` array; only the PO-number check returns a real 400 |
| 10 | `orderFinalisation.js` header | Credit is listed as a migration target but has not been migrated; the migration must carry over `po_number`, the `total_price` re-stamp and `_tryAutoAssignUser` |

---

## 8. File index

| Concern | Path |
|---------|------|
| Status / payment constants | `app/Utils/constants.js:1-24` |
| Status lookup seed | `database/seeds/OrderStatusSeeder.js` |
| Payment lookup seed | `database/seeds/PaymentStatusSeeder.js` |
| Orders schema | `database/migrations/1571924481804_order_schema.js` |
| `po_number` column | `database/migrations/1771505559711_add_po_number_to_order_schema.js` |
| Changelog schema | `database/migrations/1697529756485_orders_changelog_schema.js` |
| Order model / relations | `app/Models/Order.js` |
| Order creation | `app/Controllers/Http/OrderController.js:487` |
| Quote → order creation | `app/Controllers/Http/BasketQuoteController.js:1795` |
| **Credit payment** | `app/Controllers/Http/CreditPaymentController.js` |
| Shared finalisation helper (unused by credit) | `app/Utils/orderFinalisation.js` |
| SAP credit profile query | `app/Services/SAPService.js:4-8` |
| Admin status change | `app/Controllers/Http/OrderStatusController.js:37` |
| Admin assignment | `app/Controllers/Http/AssignUserToOrderController.js` |
| Cart middleware | `app/Middleware/VerifyCart.js`, `app/Middleware/VerifyQuoteCart.js` |
| Auth middleware | `app/Middleware/ApiTokenAuth.js`, `start/kernel.js:56` |
| Event wiring | `start/events.js:3,7` |
| Order emails | `app/Listeners/Order.js:10` (`send`), `:194` (`sendAssignment`) |
| Credit email component | `resources/views/emails/order/credit-payment.edge` |
| Email payment branching | `resources/views/emails/order/order.edge:81-90` |
| Assignment email | `resources/views/emails/orderAssigned.edge` |
| Notifications | `app/Services/NotificationService.js`, `database/seeds/NotificationTypeSeeder.js` |
| Routes | `start/routes/public.js:49,139`, `start/routes/admin.js:4-12` |
