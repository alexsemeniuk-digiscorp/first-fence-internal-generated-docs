# Paying on a Trade Credit Account — Designer's Guide

A plain-English companion to
[`order-and-payment-status-flow-trade-credit.md`](./order-and-payment-status-flow-trade-credit.md)
(the technical version). No code or database detail here — but it does cover what happens
behind the scenes, because a few of those hidden steps explain UI behaviour that otherwise
looks arbitrary.

---

## 1. What "paying on credit" means here

Some First Fence customers are **trade accounts**. Instead of paying by card, they have a
pre-agreed credit limit and get invoiced later. Checking out on credit is basically:
*"put this on my account."*

Two things make it different from every other payment method in the app:

1. **It's instant.** No card form, no 3D Secure step, no bank-transfer waiting period, no
   third-party screen. One tap and the order is confirmed.
2. **A PO number is mandatory.** The customer's own internal purchase-order reference. No
   other payment method in the app asks for anything extra.

---

## 2. Where an order comes from

Useful context, because the order **already exists** before any payment happens.

When a customer finishes the details-and-delivery part of checkout, we create the order
straight away in a **Created** state — no payment method attached yet. Paying is a second,
separate step that updates that existing order.

Practical consequences for you:

- An abandoned checkout leaves a real order sitting in **Created**. Our team can see these,
  and they're the basis of abandoned-basket follow-up. So "Created" is not a bug state.
- Payment screens act on an order that's already been created, which is why the app can
  offer a "choose a different payment method" retreat after a failed credit attempt — the
  order survives.

There are **two routes into a credit payment**, and they behave the same way at the payment
step:

| Origin | How it starts |
|---|---|
| **Normal checkout** | Customer builds a basket in the app and checks out |
| **Quote conversion** | A quote the sales team prepared is turned into an order, then paid |

---

## 3. Who can use it

Credit payment should only appear when **all** of these are true:

- The customer is **signed in**. Guests can pay by card, bank transfer or PayPal — never on
  credit.
- Their app account is **linked to a trade account** in our finance system. This link is set
  up on our side, not by the customer.
- That trade account has a **credit limit** on it.

If any of these fail, credit shouldn't be offered — and if it is shown anyway, tapping it
returns one of the errors in §6.

> There's currently **no "apply for a credit account" path in the app**. Setting up a trade
> account happens offline with the sales team. A customer without credit has nowhere to go
> in-app, which is worth designing around rather than ignoring.

---

## 4. Showing the credit balance

The app can ask for the customer's credit summary at any point. It returns the **company
name**, the **credit limit**, and the **available balance**.

### How the available balance is worked out

Conceptually: *credit limit, minus everything they currently owe us.* "Owe us" is three
things added together — invoices outstanding, goods delivered but not yet invoiced, and
orders placed but not yet delivered. It's read live from the finance system every time,
never stored or cached in the app.

You don't need the mechanics, but you do need this: **it's one number with three moving
parts behind it**, so it can change between two screens in the same session for reasons the
customer won't understand. Don't build UI that treats it as stable within a session.

### Three cases to handle

| Situation | What comes back | Design note |
|---|---|---|
| Has a credit account | Company name, limit, available balance | Normal case |
| No credit account | Only a "no credit account" flag — **no limit, no balance, no company name** | Don't assume the numbers are always present |
| **Negative** available balance | A negative number | Genuinely possible — they're over their limit. Don't clamp to zero or hide it; design a clear "over limit" treatment |

### Why the balance looks stale after ordering

This one matters. When a credit order is placed in the app, **it does not automatically
reach our finance system.** Someone on our team books it in manually afterwards and records
the finance-system order number against it.

Until that manual step happens, the customer's available balance still shows the figure from
*before* their order. So:

- **Don't** design a "your balance after this order will be £X" projection.
- **Don't** design a balance figure that appears to update the instant an order is placed —
  it would be wrong for a while, then silently correct itself.
- Showing the current available balance, labelled as the current figure, is safe.

A knock-on effect: a customer can place several orders in quick succession that each pass the
credit check individually but together exceed their limit, because none of them have
registered yet. That's accepted business behaviour, not something the UI can prevent — but
it means the credit check is a softer guarantee than it looks.

---

## 5. The happy path

```
  Cart / Basket
       │
       ▼
  Checkout — details, delivery        ← order is created here, unpaid
       │
       ▼
  Choose payment method
       │  ("Pay on account" / "Trade credit")
       ▼
  ┌─────────────────────────────────┐
  │  PO number  ← required          │
  │  Available balance shown        │
  │  Order total (inc VAT) shown    │
  │  [ Place order on account ]     │
  └─────────────────────────────────┘
       │
       ▼   instant — no third-party screen, no waiting state
  Order Confirmed
```

### The PO number field

- **Required.** Blank is rejected, and so is spaces-only.
- **Free text.** No format rule, no length limit, no validation pattern. Whatever they type
  is what appears on their paperwork and in their confirmation email.
- **Nothing is remembered.** There is **no saved or default PO number** anywhere today — not
  in profile settings, not carried over from the last order. Every order means retyping it.

> **Note:** a PO number is also collected in one other place — when a customer asks the sales
> team to raise an order from a saved quote. That flow doesn't place an order at all; it
> sends a message to the sales team with the PO in it, and a human picks it up. Same field,
> completely different outcome. Worth keeping the two visually distinct so customers don't
> expect an instant confirmation from the quote one.

### About the totals

Prices are always **recalculated on our side at the moment of payment**, from the actual
basket contents — never taken from what the app sends. If the basket or delivery changed
since the total was last displayed, the recalculated figure wins.

So the total on the payment screen should be freshly fetched rather than carried along from
an earlier step, and the confirmation screen should show the figure that came back from
placing the order, not the one the app was holding.

### The confirmation

The order becomes **Completed** immediately — there's no processing or awaiting-payment
state to render. The customer also gets a confirmation email that:

- says the order has been placed on their credit account
- reads their **PO reference** back to them
- shows the **total payable including VAT**
- deliberately has **no payment progress bar** (card and bank-transfer emails do have one,
  because those have something to wait for — credit doesn't)

Matching that in-app — PO referenced back, VAT-inclusive total, no progress indicator —
will feel consistent.

---

## 6. What happens behind the scenes when they tap

None of this is visible to the customer, but it explains a couple of UI behaviours and it's
useful when you're talking to the team about the flow.

The moment a credit order is placed, in this order:

1. **The credit check runs** — live lookup of the trade account and its balance, compared
   against the recalculated order total including VAT.
2. **The order is marked Completed** and stamped with the PO number and the payment method.
3. **The basket is closed off** so it can't be reused.
4. **Our internal team is notified** — in-app, and by email or push depending on each staff
   member's own notification preferences.
5. **The customer's confirmation email goes out**, copied to internal sales inboxes.
6. **An account manager is auto-assigned.** We look up who owns that trade account in our
   CRM, match them to a staff account, attach them to the order, and email them a
   "Credit Order Assigned" notice showing the PO number.

Steps 4–6 are all **best-effort**. If the CRM is unreachable or an email bounces, the order
still succeeds — the customer sees a normal confirmation either way. So the app should never
wait on, or report on, any of that.

Two design-relevant consequences:

- **Assignment is invisible and automatic.** The customer has an account manager attached to
  their order without ever being told. If you're designing an order-detail screen, "who's
  handling this" is information that exists — a product decision whether to surface it.
- **Closing the basket can fail on its own.** It's a separate step from completing the order,
  so in a rare glitch a customer could land on a confirmation screen with their old basket
  still populated. Worth making sure the confirmation screen clears or re-fetches basket
  state rather than trusting what it was holding.

---

## 7. Every way it can fail

All of these are **recoverable, inline errors** — not crashes, not dead ends. In every case
the order is left untouched and unpaid, so the customer can fix the problem or back out and
pick a different payment method. The flow must always allow that retreat.

| What went wrong | Message the app receives today | Design suggestion |
|---|---|---|
| PO number empty or spaces only | *"PO number is required."* | Inline field validation — catch it before the button is tappable |
| Signed in, but no trade account linked | *"No trade account found."* | Ideally unreachable: don't offer credit to these customers |
| Trade account has no credit limit | *"No credit account available."* | Same — filter earlier if you can |
| Finance system unreachable | *"Unable to retrieve credit account details."* | Temporary system problem: "Try again shortly, or choose another payment method" |
| Total exceeds available credit | *"Insufficient credit balance to complete this order."* | **The only one a real customer is likely to hit.** Deserves a designed state — see below |
| Something wrong with the order itself | *"Invalid Order ID!"* | Genuine system fault; generic error treatment is fine |

That wording is the current backend copy. It's blunt and none of it was written for a mobile
UI — treat it as raw material and rewrite it, or map each case to your own copy.

### The insufficient-credit case

This is the realistic failure. At that moment we know both the order total and the available
balance, so the screen can be helpful rather than just refusing:

- show the shortfall (*"£420 over your available credit"*)
- offer the alternatives — pay by card, or contact the account manager
- keep the basket intact

**There is no partial payment.** A customer can't put part of an order on credit and part on
a card, and there's no "request a temporary limit increase" path in the app. Please don't
design an interface that implies either exists.

---

## 8. Order statuses

Orders carry a status. Some are meaningful to customers; some are purely internal
bookkeeping. You need labels for the first group and a safe fallback for the rest.

**Statuses a customer can genuinely land on:**

| Status | What it means | Which methods reach it |
|---|---|---|
| **Created** | Order exists, not paid for. In-progress or abandoned checkout. | All — the starting point |
| **Completed** | Paid and confirmed. | Credit (always), card, PayPal, iwoca |
| **Pending** | Waiting for money to arrive. | Bank transfer, iwoca |
| **Failed** | Payment declined. | Card, iwoca |
| **Cancelled** | Customer backed out at the payment provider. | PayPal |
| **Refunded** | Money returned. Set by hand by our team, possibly long after the order. | Any |

**A trade credit order only ever goes Created → Completed.** It can never be *Pending* or
*Failed* — there's nothing to decline or wait for. Only later manual intervention by our team
can move it anywhere else.

**Internal statuses:** our staff can set any of these on any order at any time, including
long after the customer has been confirmed — *closed, sales, requested, lead, duplicate,
spam, test*, plus a state where the status has been cleared entirely.

> **Please design a fallback.** These are internal words, not customer-facing language, and
> new ones get added over time. An order-history row that renders whatever raw status the
> system holds will eventually show a customer the word "spam" or "duplicate". Safest
> approach: explicitly style the six customer-facing statuses above, and render everything
> else as a single neutral catch-all like "Processing" — never pass the raw value through.

---

## 9. What happens to the order afterwards

The customer's involvement ends at confirmation, but the order keeps moving internally. This
is why an order's status can change days later without the customer doing anything.

- **Staff can change the status manually at any time**, to any value including the internal
  ones. Every change is recorded in an edit history with who did it and what it changed from
  and to, and it notifies the rest of the team.
- **Refunds are entirely manual.** There's no automated refund path — "Refunded" appears
  because a person set it.
- **The finance-system booking** described in §4 happens here, which is when the customer's
  available balance finally reflects their order.
- **Reassignment** to a different account manager can happen, with the same edit-history
  trail.

Design implication: **order status is not a one-way progression and it isn't driven by the
customer.** If order history or an order-detail screen caches status, it'll go stale. Design
for re-fetching, and avoid progress-bar or stepper metaphors that imply orders only ever move
forward — a Completed order can become Refunded or Cancelled.

---

## 10. How credit differs from the other payment methods

Useful when designing the payment-method picker and the screens behind it.

| | Trade credit | Card | Bank transfer | PayPal |
|---|---|---|---|---|
| Must be signed in | **Yes** | No | No | No |
| Extra required field | **PO number** | — | — | — |
| Leaves the app / extra step | No | Yes (3D Secure) | No | Yes |
| Waiting or processing state | **None** | Brief | Long — awaiting funds | Brief |
| Can fail after submitting | Only account / credit-limit checks | Yes, declines | No | Yes, cancellation |
| Status after ordering | Completed | Completed or Failed | **Pending** | Completed or Cancelled |
| Auto-assigns an account manager | **Yes** | No | No | No |
| Confirmation email style | Account wording + PO, **no progress bar** | Progress bar | Bank details + progress bar | Progress bar |

Design takeaway: credit is by far the **shortest and most certain** checkout in the app —
one required field, one tap, immediate confirmation. Let it feel that fast rather than adding
reassurance states it doesn't need.

---

## 11. Open questions for the team

Things the current system doesn't decide for you, worth settling before build:

1. **Saved PO numbers** — accept retyping every time, pre-fill from the last order, or a
   proper saved default in settings? The last option needs backend work.
2. **Customers without credit** — hide the option entirely, or show it disabled with an
   explanation? And is there anything to point them at, given there's no in-app application
   path?
3. **Where the balance lives** — checkout only, or somewhere persistent like a profile or
   dashboard? How prominent should an over-limit balance be?
4. **Insufficient-credit screen** — just the refusal, or shortfall plus alternatives plus a
   route to the account manager?
5. **Status labels** — customer-facing wording for the six statuses in §8, plus the neutral
   catch-all for everything internal.
6. **Surfacing the account manager** — the assigned person is known but never shown. Should
   an order-detail screen introduce them?
7. **Order history** — should credit orders be visually distinct from card orders, e.g. with
   the PO reference on the row?