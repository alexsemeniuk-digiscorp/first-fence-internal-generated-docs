# Paying on a Trade Credit Account — Designer's Guide

A plain-English companion to
[`order-and-payment-status-flow-trade-credit.md`](./order-and-payment-status-flow-trade-credit.md)
(which is the technical version). No code here — just what the app does, what the customer
sees, and the states you need to design for.

---

## 1. What "paying on credit" means here

Some First Fence customers are **trade accounts**. Instead of paying by card, they have a
pre-agreed credit limit and get invoiced later. Checking out on credit is basically:
*"put this on my account."*

Two things make this different from every other payment method in the app:

1. **It's instant.** There's no card form, no 3D Secure step, no bank-transfer waiting
   period, no third-party screen. The customer taps once and the order is done.
2. **A PO number is mandatory.** This is the customer's own internal purchase-order
   reference. Every other payment method in the app has no such field.

---

## 2. Who can use it

Credit payment should only appear as an option when **all** of these are true:

- The customer is **signed in**. (Guests can pay by card, bank transfer or PayPal — but not
  on credit.)
- Their account is **linked to a trade account** in our finance system.
- That trade account has a **credit limit set**.

If any of those aren't true, credit shouldn't be offered — or if it is shown, tapping it
returns one of the errors in §5.

> **Worth designing for:** a signed-in retail customer and a signed-in trade customer see
> different payment options. There's currently no "apply for a credit account" path in the
> app. If a customer without credit somehow reaches this option, the only message they get
> is a fairly blunt "No credit account available."

---

## 3. Showing the credit balance

The app can ask for the customer's credit summary at any time. It comes back as:

- **Company name**
- **Credit limit** — the total they're allowed to owe
- **Available balance** — what's left to spend right now

Three things to handle in the design:

| Situation | What you get back | Design note |
|---|---|---|
| Has a credit account | Company name, limit, available balance | Normal case |
| No credit account | Just a "no credit account" flag — **no limit, no balance, no company name** | Don't build a layout that assumes the numbers are always there |
| Available balance is **negative** | A negative number | Genuinely possible — the customer is over their limit. Don't clamp it to zero or hide it; design a clear "over limit" treatment |

**Important caveat on freshness:** the available balance comes live from the finance system,
but a newly placed app order does **not** immediately reduce it. There's a manual step on
our side before the order registers against the account.

So please **don't** design a "your balance after this order will be £X" projection, or a
balance figure that appears to update the moment an order is placed — it would be wrong for
a while and then silently correct itself. Showing the current available balance, clearly
labelled as the current figure, is safe.

A practical consequence: a customer can place several orders in quick succession that each
pass the credit check but together go over their limit. That's a known business behaviour,
not something the UI can prevent.

---

## 4. The happy path

```
  Cart / Basket
       │
       ▼
  Checkout — details, delivery
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
       ▼   (instant — no third-party screen, no waiting state)
  Order Confirmed
```

### About the PO number field

- **Required.** Blank won't be accepted, and neither will spaces-only.
- **Free text.** There's no format rule, no length limit, no validation pattern. Whatever
  the customer types is what appears on their paperwork.
- **Nothing is remembered.** There is currently **no saved or default PO number** anywhere —
  not in profile settings, not carried over from a previous order. Every order means typing
  it again from scratch.

> **Design opportunity worth raising:** the "type it again every time" behaviour is a real
> friction point for repeat trade buyers, and there's no backend support for a default PO
> today. Options range from cheap (pre-fill from the customer's most recent order) to a
> proper saved default in profile settings — the latter needs backend work. Similar
> per-customer defaults already exist for other things, so it's a plausible ask.

### About the confirmation

Once placed, the order is **immediately confirmed** — no "processing" or "awaiting payment"
state. The customer also gets a confirmation email that:

- says the order has been placed on their credit account
- shows their **PO reference** back to them
- shows the **total payable including VAT**
- deliberately has **no payment progress bar** (card and bank-transfer emails do have one,
  because those have something to wait for — credit doesn't)

Keeping the in-app confirmation screen consistent with that — reference back the PO number,
show the VAT-inclusive total, no progress indicator — will feel right.

Behind the scenes, an account manager may be automatically assigned to the order. That's
entirely internal; the customer never sees it and it doesn't affect the UI.

---

## 5. Every way it can fail

All of these are **recoverable, inline errors**, not crashes or dead ends. In every single
case the order is left untouched and unpaid, so the customer can fix the problem or **back
out and choose a different payment method**. Please make sure the flow always allows that
retreat.

| What went wrong | Message the app receives today | Design suggestion |
|---|---|---|
| PO number empty or just spaces | *"PO number is required."* | Inline field validation — catch it before the customer taps the button |
| Signed in, but no trade account linked | *"No trade account found."* | Ideally never reachable: don't offer credit to these customers |
| Trade account has no credit limit | *"No credit account available."* | Same — filter this out earlier if possible |
| Can't reach the finance system | *"Unable to retrieve credit account details."* | Treat as a temporary system problem: "Try again shortly, or choose another payment method" |
| Order total exceeds available credit | *"Insufficient credit balance to complete this order."* | **The most likely real-world failure.** Deserves a proper designed state, not a toast — see below |
| Something wrong with the order itself | *"Invalid Order ID!"* | A genuine system fault; generic error treatment is fine |

The wording above is the current backend copy. It's blunt and none of it was written for a
mobile UI — treat it as raw material and rewrite it, or map each case to your own copy.

### The insufficient-credit case deserves attention

This is the one a real customer will actually hit. When it happens we know the order total
and the available balance, so the screen can be genuinely helpful rather than just refusing:

- show the shortfall (*"£420 over your available credit"*)
- offer the alternatives — pay by card, or contact the account manager to discuss the limit
- keep the basket intact

There is **no partial payment**. A customer can't put part of an order on credit and part on
a card. Please don't design an interface that implies they can.

---

## 6. Order statuses — what customers can see

Orders carry a status. Some are meaningful to customers, some are purely internal admin
bookkeeping. You'll need labels for the first group and a safe fallback for the rest.

**Statuses a customer can genuinely land on:**

| Status | What it actually means | Which payment methods reach it |
|---|---|---|
| **Created** | Order exists but hasn't been paid for. An abandoned or in-progress checkout. | All — this is the starting point |
| **Completed** | Paid and confirmed. | Credit (always), card, PayPal, iwoca |
| **Pending** | Waiting for money to arrive. | Bank transfer, iwoca |
| **Failed** | Payment was declined. | Card, iwoca |
| **Cancelled** | Customer backed out at the payment provider. | PayPal |
| **Refunded** | Money returned. Set manually by our team, possibly long after the order. | Any |

**A trade credit order only ever goes Created → Completed.** It can never be *pending* or
*failed*, because there's nothing to decline or wait for. If our team later intervenes it
could become *refunded*, *cancelled* or one of the internal statuses below.

**Internal statuses** — our staff can set any of these on any order at any time, including
long after the customer has been confirmed: *closed, sales, requested, lead, duplicate,
spam, test*, and a state where the status has been cleared entirely.

> **Please design a fallback.** These internal values are not customer-facing language and
> new ones get added over time. An order-history row that renders whatever raw word the
> system holds will eventually show a customer the word "spam" or "duplicate". Safest
> approach: explicitly style the six customer-facing statuses above, and show everything
> else as a single neutral catch-all (e.g. "Processing" or "In progress") rather than passing
> the raw value through.

---

## 7. How credit differs from the other payment methods

Useful if you're designing the payment-method picker and the screens behind it.

| | Trade credit | Card | Bank transfer | PayPal |
|---|---|---|---|---|
| Must be signed in | **Yes** | No | No | No |
| Extra required field | **PO number** | — | — | — |
| Leaves the app / extra step | No | Yes (3D Secure) | No | Yes |
| Waiting or processing state | **None** | Brief | Long — awaiting funds | Brief |
| Can fail after submitting | Only credit-limit / account checks | Yes, declines | No | Yes, cancellation |
| Status after ordering | Completed | Completed or Failed | **Pending** | Completed or Cancelled |
| Confirmation email style | Account wording + PO, **no progress bar** | Progress bar | Bank details + progress bar | Progress bar |

Design takeaway: credit is by far the **shortest and most certain** checkout in the app.
One required field, one tap, immediate confirmation. It's worth letting it feel that fast
rather than adding reassurance states it doesn't need.

---

## 8. Open questions for the team

Things the current backend doesn't decide for you, and that are worth settling before build:

1. **Saved PO numbers** — accept retyping every time, pre-fill from the last order, or build
   a proper saved default? The third option needs backend work.
2. **Customers without credit** — hide the option entirely, or show it disabled with an
   explanation and a route to apply?
3. **Where the balance appears** — only at checkout, or also somewhere persistent like a
   profile or dashboard? And how prominent should an over-limit (negative) balance be?
4. **Insufficient-credit screen** — how much detail? Just the refusal, or shortfall +
   alternatives + a way to contact the account manager?
5. **Status labels** — the customer-facing wording for the six statuses in §6, plus the
   catch-all label for everything internal.
6. **Order history** — should credit orders be visually distinguished from card orders
   (e.g. showing the PO reference on the row)?