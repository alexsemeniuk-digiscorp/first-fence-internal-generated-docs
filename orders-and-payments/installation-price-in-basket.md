# "Installation" Pricing in the Basket — Why the Numbers Don't Add Up

Investigation into a report from the design team:

> *"Can you check how 'Installation' is calculated in the basket? If I add Installation to a
> product, it doesn't show the price of that Installation for that specific product — it shows a
> general one. But if I add another product and add Installation to it as well, the total for that
> service either doesn't change at all, or comes out slightly higher — not the one specified when
> I chose it."*

**The report is accurate.** Every part of it has been reproduced. This document explains why.

---

## TL;DR — in plain language

**The designer found a real problem, and it is not a display glitch. The product page and the
basket use two different calculators that follow different rules, so their numbers can never be
added up.**

Four things are going on:

**1. Installation has a minimum charge, and it's large.**
Every fence installation type has a minimum price. For fencing sold by the metre that minimum is
**£2,000**. A short run of fence only costs a few hundred pounds to install, so the minimum kicks
in and the customer is shown £2,000. The same happens for a railing kit, a mesh kit, a timber kit —
they all show **exactly £2,000**. You have to order roughly **35–40 metres** before the real
per-metre rate becomes bigger than the minimum and you start seeing a genuine, product-specific
price. So the "general price" the designer noticed is the minimum charge, showing through.

**2. The minimum is charged once for the whole basket, not once per product.**
The product page shows each product's installation *already raised up to the £2,000 minimum*. The
basket, however, adds up the products' **real** installation costs (ignoring the minimum) and only
then applies **one single £2,000 minimum** to the whole order. So:

- Add a second product while the basket is still under £2,000 → **the total doesn't move at all.**
  The customer was shown £2,000 for that second product and the basket adds £0.
- Add a second product that pushes the basket over £2,000 → the minimum stops applying to
  everything, and the total jumps to the real combined cost. It goes **up a little**, not by the
  amount the customer was shown. In our test, adding a service quoted at £2,000 raised the basket
  by **£330**.

That is exactly the designer's sentence: *"either does not change at all, or shows up as slightly
higher."*

**3. The price shown when you choose Installation is thrown away.**
When you add to basket, only your *answers* are saved — ground type, the questions you ticked,
the metres you typed. The price itself is never sent or stored. The basket works it out again from
scratch, with the different rules above. So the basket literally cannot show you "the price that
was specified when I chose it" — it doesn't have it.

**4. The basket deliberately shows only one combined figure.**
No basket line shows an installation price at all. Each line just says *"Installation Included
(See summary for combined pricing)"* and the single total appears in the summary panel on the
right. So there is nothing for the customer to check their per-product price against.

**What this means:** the product page promises a per-product price; the basket delivers an
order-wide price. Both are "working as written", but together they make **contradictory promises to
the customer**, and the customer sees the contradiction. This needs a product decision (§8), not
just a code fix.

**Separately, we found several genuine bugs** while investigating — including one where a
customer's installation setup is silently thrown away and replaced with a generic default, and one
where the trade credit-limit check under-counts the order. Those are in §6 and can be fixed without
waiting for the design decision.

---

## 1. Reproduced, with real production numbers

Two engineers independently extracted the live pricing code and ran it against the real
production database. Using **868 Mesh** panels (Soft Dig, £51.79/m, £2,000 minimum):

| Basket contents | Price shown on each product page | Basket "Installation" total |
|---|---|---|
| 15 m, installation on it | `+ £2000.00` | **£2,000.00** |
| 15 m **+ 18 m**, installation on both | `+ £2000.00` and `+ £2000.00` | **£2,000.00** ← *unchanged* |
| 20 m, installation on it | `+ £2000.00` | **£2,000.00** |
| 20 m **+ 25 m**, installation on both | `+ £2000.00` and `+ £2000.00` | **£2,330.55** ← *"slightly higher"* |

A third engineer reproduced the same behaviour with Palisade Fencing (£57.15/m) and a gate
(flat £210, £1,000 minimum): 1 bay palisade + 1 gate → basket stays at **£2,000**, so a service
quoted at £1,000 adds **nothing**.

And the reverse case — the basket can **undercharge**. A product with a £250 minimum (quoted
£250) added to a product with a £5,000 base: the expected £5,250 becomes **£5,100**, because the
first product now contributes its real £100 base instead of its £250 minimum.

---

## 2. The root cause: two calculators, different rules

Both live in `gatsby-website/src/utils/product/installation-calculations.js`.

| | Product page (what the customer is quoted) | Basket (what the customer is charged) |
|---|---|---|
| Function | `getTotalInstallPrice` (`:122-150`) | `getCombinedInstallPrice` (`:152-271`) |
| Called from | `src/models/InstallationType.js:164` | `shopping-cart-manager.js:54,83,132,179,226` |
| Minimum charge | **applied** — `ignoreMinCharge = false`, so `basePrice = basePrice < minCharge ? minCharge : basePrice` (`:138-140`) | **max across all products** (`:287-288`), applied **once to the whole basket** |
| Base price | this product only | **sum of all products' un-floored bases** (`:163`) |
| Above the minimum | n/a | recomputes **every** product with `ignoreMinCharge = true` (`:192-199`) — every product loses its floor |
| Surcharges | all of them, on this product's base | de-duplicated across products (see §4.2) |

The decisive line is `:189`:

```js
if (preMinChargeBasePrice >= minCharge) {
  // every product recalculated with ignoreMinCharge = true
```

Below that threshold the whole basket collapses to a single `minCharge`. Above it, every
product's floor evaporates. **The quoted figures are therefore un-summable by construction**, and
crossing the threshold produces a discontinuity of hundreds of pounds.

### Why the "general price" appears at all — the minimum charge distribution

From the live `cdn.installationtypes` collection (42 rate cards, 888 products offer installation):

| Minimum charge | Priced by | Products |
|---|---|---|
| £1,000 | quantity | 567 |
| £1,900 | quantity | 135 |
| **£2,000** | **metres** | **127** ← all metre-priced fencing |
| £0 | quantity | 46 |
| £3,000 | quantity | 12 |

Only **five distinct minimum charges exist across the whole catalogue**, and every metre-priced
fencing product shares the same £2,000. At £51.79–57.15/m you need ~35–40 m before the real rate
exceeds it. Below that, the product-specific rate is invisible **by design** — which is precisely
what the designer described.

Several rate cards are also byte-identical duplicates of each other (`V Mesh` = `Safe Top Mesh` =
`Railings`; `868 Mesh` = `Stripe Mesh`; `Crash Barrier` = `Timber`), reinforcing the impression of
a single global price.

---

## 3. The quoted price is never saved

`src/models/InstallationType.js:267-277` — the add-to-basket payload carries **selections only**:

```js
{ id, sqlId, selectedInstallationPrice, additionalInfo,
  surcharges: [{ id, selectedOption, modifierQuantity }] }
```

No price, no quantity, no meterage. Confirmed on the server side too: `installation_types` and
`installation_surcharges` have **no price column**
(`database/migrations/1660203619602_...:8-16`, `1660205225659_...:8-21`), and `website-api` performs
**no installation arithmetic at all** — a full grep of `app/` finds only pass-through reads.

Consequences:

- The basket cannot display "the price shown when I selected it" — it was never transmitted.
- Prices are re-derived from **live** Mongo config on every basket render, so an admin editing a
  minimum charge or a rate band **silently reprices every open basket**.
- **No server-side validation exists.** `OrderController.js:501-502` reads `installationPrice` and
  `installationRemovalPrice` straight from the request body and writes them verbatim at `:584-585`;
  `BasketQuoteController.js:731,774-775` does the same. A crafted POST can set any installation
  price. There is no validator (`app/Validators/` has nothing for installation).

---

## 4. Three more reasons the basket disagrees with the product page

### 4.1 Removal charges are silently moved to a different row

Any surcharge whose selected option is priced `PRICE_PER_METER` is reclassified as a "removal"
charge (`installation-calculations.js:339-341`) and reported in a **separate bucket**, stored in a
**different database column** (`orders.installationRemovalPrice`).

The product page folds it into one figure; the basket splits it:

| | Shown |
|---|---|
| Product page | `+ £2,300.00` (one figure, includes £300 removal) |
| Basket "Installation" row | **£2,000.00** |
| Basket "Removal" row | £300.00 |
| Basket "Installation Subtotal" | £2,300.00 |

So comparing the like-named row shows a £300 shortfall. Only the *Subtotal* reconciles.

### 4.2 The second product's surcharge answers get discarded

Below the minimum charge, surcharges are merged through `combineSurcharges` (`:356-372`), which
**de-duplicates by surcharge id and keeps only the highest `modifier`** (`compareSurchargeOptions`,
`:381-385`). All 42 installation types reference **the same 8 surcharge documents**, so collisions
are guaranteed.

Result: the second product's answers are either thrown away (total unchanged) or *silently
upgrade* the first product's answer to the dearer option (total slightly higher). Reproduced:
product A quoted £500 with a £100 option + product B quoted £800 with a £400 option → basket
£1,200, with A silently upgraded to B's £400 option.

Note this collapse happens **only** in the under-minimum branch; above it surcharges are charged
per product. The behaviour is inconsistent by branch.

Two further defects here:

- `compareSurchargeOptions` compares `modifier` **without comparing `unit`** — so a `PERCENTAGE 50`
  loses to a `PRICE 25` because `50 > 25` is evaluated across incompatible units. Verified: a
  50%-of-base surcharge was replaced by a flat £25.
- `oncePerOrder` percentage surcharges are applied to the **combined** base (`:232-235`), so a
  percentage belonging to product A is also charged on product B's value. Reproduced: A quoted
  £240 (£200 + 20%) + B quoted £1,000 → basket **£1,440**, not £1,240.

### 4.3 Adding a *cheap* product can raise the whole order

Because `minCharge` is the **maximum** across products (`:287-288`), adding a low-value product
whose type carries a £900 minimum lifts the entire order's floor to £900 — an increase unrelated
to anything that product displayed.

---

## 5. Is this a bug, or intended?

**The global figure is intended.** The code says so in two places:

- `card-item/installation-check/installation-check.js:28-33` renders
  `Installation Included` / `(See summary for combined pricing)`
- `features/checkout/sidebar/installation/Installation.js:43` warns
  `"Combined installation pricing may differ."`

Storage agrees: there is **one** `orders.installationPrice` column for the whole order,
`order_details` has no installation column, and the cart-line `products` table has none either. The
admin panel likewise shows a single figure (`admin-website-v2/src/pages/orders/order-page/order-map.js:120-124`).

**So a per-line installation price cannot be displayed today even if someone wanted it** — no data
structure can hold one.

**What is *not* intended** is the contradiction: the product page shows a per-product price *and*
folds installation into its headline "Add To Basket" total (`src/templates/product/product.js:356`
passes `includeInstallation = true`). It promises something the basket then refuses to honour. That
is a genuine spec conflict, and it is what the designer walked into.

---

## 6. Genuine bugs found (fixable independently of the design decision)

### 6.1 🔴 A single surcharge is never saved — and installation silently reverts to a generic default

`website-api/app/Utils/installation.js:24`:

```js
if (Array.isArray(surcharges) && surcharges.length > 1) {
```

Should be `> 0`. With **exactly one** surcharge, no `installation_surcharges` rows are written at
all. The damage then cascades:

1. On read, `getInstallationSurcharges` requires `surchargeIds.length === sqlSurcharges.length`
   (`installation.js:117-129`), gets `1 !== 0`, and returns `null`.
2. `getInstallationType` therefore returns `null`.
3. `CartController.js:78-83` then substitutes **the product's default installation type straight
   from Mongo**, with no selections and no `sqlId`.
4. The client sees an invalid config, `requiresInstallation` becomes false, and the product drops
   out of the calculation entirely.
5. With no `sqlId`, re-saving from the cart calls `createInstallationType` again
   (`shopping-cart-manager.js:139-142`), inserting a **duplicate** `installation_types` row for the
   same product — of which `hasOne` returns only one.

This is a **second, independent mechanism** producing "it shows a general one", and it is silent —
`CartController.js:78-83` logs nothing, even though a `Grafana.emerg` hook sits two lines below.

**It also cannot self-heal:** `InstallationTypeController.update` (`:144-165`) only merges
*existing* surcharge rows and returns `null` when they're absent — it never creates missing ones.

### 6.2 🔴 The basket is the only read path with strict surcharge matching

Every other consumer passes `ignoreSurchargeMismatch = true` — `OrderController.js:800-804`,
`BasketQuoteController.js:80-84` and `:852-856`, `Portal/CustomerOrderController.js:89-93`,
`Portal/CustomerQuoteController.js:93-97`. **Only `CartController.index` and
`mapProductsInstallation` use the strict default `false`.**

Someone papered over this mismatch everywhere except the basket. So if an admin adds or removes a
surcharge on an installation type in Mongo after a customer configured it, installation silently
reverts to the generic default **on the basket page only** — order and quote pages still render
correctly.

### 6.3 🟠 Surcharge writes are fire-and-forget (race condition)

`website-api/app/Utils/installation.js:25-32` uses `surcharges.forEach(async (s) => { await ... })`
— the `async` callbacks are **never awaited**. `saveInstallationType` returns, and the HTTP 200 with
`sqlId` is sent, **before the surcharge rows exist**. Any basket refresh in that window lands in
the count-mismatch path above. This matches an intermittent "installation went generic".

Fix: `await Promise.all(surcharges.map(...))`.

### 6.4 🔴 The credit-limit check under-counts orders with a removal charge

`app/Controllers/Http/CreditPaymentController.js:174-179` adds `installationPrice` into
`totalForComparison` but **never references `installationRemovalPrice` at all** — it is the only
payment controller in the codebase that doesn't (`WorldpayController.js:153-157`,
`CardPaymentController.js:136-139`, `PayPalController.js:157-173` and `:358-361`, and
`app/Utils/order-helpers.js:111-117` all include it).

That total then drives the credit gate at `:182-184`:

```js
const totalWithVAT = totalForComparison * VAT;
if (availableBalance < totalWithVAT) { ... }
```

So for a trade order carrying a removal charge, the **available-balance check is performed against
a figure lower than the real order total**. An order can pass the credit check when the true total
would have exceeded the customer's available credit. This is a credit-control gap rather than a
mischarge — the customer is not billed less — but it lets orders through that should have been
stopped, and the size of the gap is the whole removal charge (£15/m, so £300 on a 20 m removal).

### 6.5 🟠 An always-true check inflates the product page total

`gatsby-website/src/utils/product/price-calculations.js:72` reads
`product?.installationType.hasValidInstallationCalc` **without calling it** — a function reference,
always truthy. Since the PDP passes `includeInstallation = true`
(`src/templates/product/product.js:356`), the headline "Add To Basket" total can include a stale
installation price for an invalid configuration.

### 6.6 🟡 Stale subtotal after a variant-quantity change

`shopping-cart-manager.js:90-106` (`updateProductVariantQty`) and `:186-205` (`onCalendarChange`)
recompute `deposit` but **never** `install` — unlike `updateProductQty` at `:83`. Since variant
quantities can change total meterage, the Installation subtotal can display a figure for the
previous meterage until some other action refreshes it. An independent "doesn't change at all".

### 6.7 🟡 Data errors in the live rate cards

- **`358 D Mesh` has Medium Dig cheaper than Soft Dig** — £55.95 vs £60.42 below 51 m, then it
  flips above. Its twin `358 Mesh` reads correctly (60.42 / 66.45). Data-entry error.
- **An `Unknown` installation type exists** (`64d25990bd460a0082c2ed7d`) with `minCharge: 1000`,
  zero prices and zero surcharges — created by the `createInstallationType` defaults
  (`cdn-graphql-v2/src/resolvers/installation-type.js:25-31`). No products point at it today; any
  that did would compute a base of 0 and a flat £1,000.
- **Surcharge lists are inconsistent across types** — 8 for fencing, 7 for most gates, 6 for
  temporary-fencing/hoarding gates, 0 for `Unknown`. Combined with §6.2, a mixed basket is walking
  into the strict-matching branch.
- Trailing whitespace in 8 type names and 6 price-group names (`"Soft Dig "`). Cosmetic today
  because matching is by `uid`, but it breaks any name-based comparison.

### 6.8 🟡 Latent: rate bands assumed pre-sorted

`installation-calculations.js:30-35` walks the `prices` array **backwards** and breaks on the first
match, which assumes it is stored ascending by `lowerBound`. Nothing sorts it — not the Mongoose
model, not the resolver, not the admin editor (`admin-website-v2/src/utils/installation-prices.js`).
Unsorted data silently picks the wrong band.

### 6.9 ⚠️ Suspected: DECIMAL columns arrive as strings

Verified against the project's own stack (knex + `mysql2@3.12`, `config/database.js:52-61`, no
`decimalNumbers` / `typeCast` option) that MySQL DECIMAL values come back as **strings**. In
`app/Utils/price-calculations.js:52-55`, `"0.00"` is a truthy non-empty string, so
`if (product.extraPrice) totalPrice += product.extraPrice` **concatenates instead of adding**. A
probe produced `priceEa*qty -> 37.5` then `+= extraPrice -> "37.50.00"`, and two products →
`"037.50.0037.50.00"`. The same class of problem would hit `order-helpers.js:89-122`
(`totalPrice = order.total_price`, then `+= installationPrice`, then `* VAT` → `NaN`).

**This does not affect the basket's installation figure** (that math runs on Mongo numbers), so it
is not the designer's symptom — but it would be a live divergence between the basket and the
order/email/payment totals. **Marked suspected, not confirmed:** the local MySQL tables are empty,
so it could not be observed end to end. Verify with one query on a real order before acting.

---

## 7. What we ruled out

| Hypothesis | Verdict |
|---|---|
| VAT applied twice, or ex-VAT vs inc-VAT mismatch | **Ruled out.** Installation is ex-VAT at both the product page (`installation-price.js:12-14`) and the basket rows (`summary/installation/installation.js:19,27,34`); one caption line adds inc-VAT (`:36`). A 20% jump would not read as "slightly", and the reproduced £2,330.55 is fully explained by the minimum-charge branch. Worth noting the presentation is still inconsistent — the PDP figure has no inc-VAT twin while the basket subtotal shows both. |
| Rounding / floating-point drift | **Ruled out as the cause.** `roundTo2Dp` (`:399-401`) is applied at every step. Sub-penny only. (Two cosmetic flaws: the `Number.EPSILON` guard is added before the `×100` so it is inert, and `safeAddition` double-rounds.) |
| `total = x` overwrite instead of `total += x` | **Ruled out.** All accumulation uses `safeAddition`. |
| Quantity multiplication missing or inconsistent | **Ruled out.** Consistently routed through `parentQuantity` / `parentMeters` (`:27-32`); verified qty 1 → £50, qty 3 → £150. |
| Installation double-counted in the goods subtotal | **Ruled out.** `summary.js:34` calls `getProductsTotalPrice` without `includeInstallation`. |
| A per-product price exists but isn't rendered | **Ruled out.** No column exists anywhere to hold one (§5). |

---

## 8. Recommendations

### First, a product decision is needed

The two views make contradictory promises, and no amount of code tidying resolves that. Pick one:

**Option A — make the basket per-product (matches what the designer expected).**
Show an installation price on each basket line, and show the minimum charge as **one explicit
order-level adjustment line** ("Installation minimum charge adjustment: +£1,632.84"). Requires
`getCombinedInstallPrice` to return a per-product breakdown, and — for persistence — a new
per-line column. Most honest to the customer, most work.

**Option B — stop promising a per-product price.**
Remove the `+ £X` from the product page (`Installation.js:91-94`), stop folding installation into
the PDP total (`product.js:356`), and replace both with a message like *"Installation is quoted
across your whole order, minimum £2,000 — see your basket for the total."* Cheapest, and removes
the contradiction, but tells the customer less.

**Option C — revisit the minimum charge itself.** Worth asking First Fence whether a £2,000
minimum *per order* is even the intended commercial rule, because the current code applies it
per-order while the product page applies it per-product. If the real rule is "minimum £2,000 per
site visit", the basket is right and the product page is wrong. **This question should be settled
before any code changes** — it determines which of A or B is correct.

Recommendation: ask about Option C first, then implement A if the answer is "per order".

### Fix regardless of the above

In order of cost-to-benefit:

1. `website-api/app/Utils/installation.js:24` — `> 1` → `> 0` (§6.1). One character, stops silent
   configuration loss.
2. `installation.js:25-32` — `await Promise.all(...)` (§6.3).
3. `CreditPaymentController.js:174-179` — include `installationRemovalPrice` in the credit check (§6.4).
4. `CartController.js:78-83` — log when the generic fallback fires (§6.1); it is currently silent.
5. Decide whether the basket should pass `ignoreSurchargeMismatch = true` like every other read
   path, or whether the other five paths are the ones that are wrong (§6.2).
6. `price-calculations.js:72` — add the missing `()` (§6.5).
7. Recompute `install` in `updateProductVariantQty` and `onCalendarChange` (§6.6).
8. `compareSurchargeOptions` — compare `unit` before `modifier` (§4.2).
9. Sort `installationPrices[].prices` by `lowerBound` on read (§6.8).
10. Fix the `358 D Mesh` dig prices and delete the `Unknown` type in the CMS (§6.7).
11. **Recompute installation server-side** at `POST /order/create` and the basket-quote endpoint,
    instead of trusting the client (§3).
12. Verify §6.9 with one query against a real order.

### Add tests

`gatsby-website/src/utils/product/__tests__/` contains only `structured-data.test.js`. The
highest-value calculation in the basket has **zero tests**. The scenarios in §1 are ready-made
cases.

---

## 9. How to reproduce (click by click)

**Symptom: "the total doesn't change at all"**

1. Open a metre-priced fencing product that offers Installation (e.g. an **868 Mesh** panel).
2. Set the quantity so the total run is about **15 m**.
3. Under Installation, click **Calculate Price**, choose ground type **Soft Dig**, answer every
   question, submit.
4. Note the figure beside the checkbox: **`+ £2000.00`**. Add to basket.
5. Open a *different* 868-Mesh-family product. Set quantity to about **18 m**. Repeat step 3 — it
   also shows **`+ £2000.00`**. Add to basket.
6. Open the basket and read the summary panel.
   **Expected by the designer: ~£4,000. Actual: £2,000.00 — unchanged.** Neither basket line shows
   any installation price, only "Installation Included (See summary for combined pricing)".

**Symptom: "slightly higher"**

7. Repeat with **20 m** and **25 m**. Each product page again shows `+ £2000.00`.
8. The basket summary now reads **£2,330.55** — not £4,000, and not the £2,000 quoted.

**Symptom: the Installation row is lower than the product page figure**

9. On one product, answer *"Removal of Existing Fencing/Gates Required?"* = **yes** and enter **20**
   metres (£15/m). The product page shows one figure, `+ £2300.00`.
10. In the basket, the **Installation** row shows **£2,000.00** and a separate **Removal** row shows
    **£300.00**.

**Symptom: installation reverts to a generic default (§6.1)**

11. Use an installation type with **exactly one** surcharge. Configure it on a product page, add to
    basket.
12. `SELECT * FROM installation_surcharges WHERE installation_type_ID = <new id>;` → **0 rows**.
13. Reload the basket: the configuration is gone and the installation figure is wrong or missing.

**Symptom: stale subtotal (§6.6)**

14. With installation set, change a **variant/option quantity** (the per-option boxes, not the main
    stepper). The Installation subtotal does not recalculate, even though the meterage changed.
    Changing the **main** quantity does refresh it.

---

## 10. What to instrument to prove it in a live environment

1. `installation-calculations.js:162-166` — log `{ minCharge, preMinChargeBasePrice, branch:
   preMinChargeBasePrice >= minCharge }` plus, per product, `{ productId, ownMinCharge, basePrice,
   totalInstallPrice }`. Any product where `basePrice < ownMinCharge` while the branch is `true` is
   a proven case.
2. `installation-calculations.js:356-372` — log every `combineSurcharges` collision with both
   modifiers and which won. Each collision is a discarded per-product surcharge.
3. `website-api/app/Utils/installation.js:111-129` — log
   `{ productId, sqlSurcharges: sqlSurcharges?.length, surchargeIds: surchargeIds?.length,
   ignoreSurchargeMismatch, returnedNull }`. Any `returnedNull` from `CartController.index` is a
   silent-fallback event.
4. `installation.js:24` — log `surcharges.length` on every save; every `=== 1` is a permanently
   lost configuration.
5. SQL health checks:
   ```sql
   -- installation configs that saved no surcharges
   SELECT it.id, it.product_ID, COUNT(s.id) AS surcharges
   FROM installation_types it
   LEFT JOIN installation_surcharges s ON s.installation_type_ID = it.id
   GROUP BY it.id HAVING COUNT(s.id) = 0;

   -- duplicate rows from the missing-sqlId re-create path
   SELECT product_ID, COUNT(*) FROM installation_types
   GROUP BY product_ID HAVING COUNT(*) > 1;
   ```

---

## 11. Code reference index

**gatsby-website — all installation pricing lives here**
- `src/utils/product/installation-calculations.js` — the whole engine:
  `calculateBaseInstallPrice` `:17-39`, surcharge math `:52-114`,
  **`getTotalInstallPrice` (product page) `:122-150`**,
  **`getCombinedInstallPrice` (basket) `:152-271`**, the branch `:189-229`,
  `oncePerOrder`/removal `:232-251`, `prepareInstallationProducts` (MAX minCharge) `:278-293`,
  `getCombinedBasePrice` `:301-317`, `separateSurcharges` `:324-347`,
  `combineSurcharges` `:356-372`, `compareSurchargeOptions` `:381-385`,
  `roundTo2Dp` `:399-401`, `safeAddition` `:408-416`
- `src/models/InstallationType.js:152-181` (quote calculation), `:267-277` (the payload)
- `src/features/product/installation/Installation.js:91-94`,
  `installation-price/installation-price.js:12-15` — the `+ £X` the customer sees
- `src/templates/product/product.js:356` — PDP total includes installation
- `src/features/shopping-cart-manager/shopping-cart-manager.js:54,83,132,179,226` — recompute sites;
  `:90-106`, `:186-205` — the two that forget
- `src/features/shopping-cart/summary/installation/installation.js:12-39` — the only price display
- `src/features/shopping-cart/card-item/installation-check/installation-check.js:28-33` — "(See
  summary for combined pricing)"
- `src/features/checkout/sidebar/installation/Installation.js:43` — "Combined installation pricing
  may differ."
- `src/utils/product/price-calculations.js:22-81` (totals), `:72` (the missing `()`)
- `src/utils/general.js:40-42,53-55` — VAT and formatting

**website-api — stores selections, computes nothing**
- `app/Utils/installation.js:24` (the `> 1` bug), `:25-32` (unawaited `forEach`),
  `:45-97` (Mongo+SQL merge), `:110-129` (strict count check), `:265-270` (nulls the type)
- `app/Controllers/Http/CartController.js:44-120`, **`:78-83` (the silent generic fallback)**
- `app/Controllers/Http/InstallationTypeController.js:65-68`, `:144-165` (cannot repair)
- `app/Controllers/Http/OrderController.js:501-502,569,584-585,800-804` — client-trusted price
- `app/Controllers/Http/BasketQuoteController.js:731,774-775`
- `app/Controllers/Http/CreditPaymentController.js:174-184` — **omits removal price from the credit check**
- `app/Utils/order-helpers.js:89-122`, `app/Utils/price-calculations.js:42-71`
- Migrations: `1660203619602_installation_types_schema.js:8-16`,
  `1660205225659_installation_surcharges_schema.js:8-21`,
  `1571924481804_order_schema.js:12-13` (the two order-level columns),
  `1571815719437_products_schema.js:8-58` (no installation column)

**cdn-graphql-v2 — the rate cards**
- `src/models/installation-type.js:3-24`, `src/schema/installation-type.js:2-29`
- `src/models/surcharge.js:3-18`, `src/schema/surcharge.js:5-9`
- `src/models/product.js:96` — `installationType: String` (a single id, no per-product override)
- `src/resolvers/product.js:842-849`, `src/resolvers/installation-type.js:25-31,109-124`

**admin-website-v2**
- `src/pages/content/installation/installation-type/edit-installation-type.jsx:160-168` — confirms
  `minCharge` is in **pounds** (`prefix={'£'}`)
- `src/pages/orders/order-page/order-map.js:120-124` — single installation figure
- `src/utils/installation-prices.js` — no sorting

**Live data** — `cdn.installationtypes` 42 docs, `cdn.surcharges` 8 docs, 888 of 7,885 products
offer installation. Local MySQL `website` tables are empty (schema-only), so all pricing evidence
comes from the Mongo production dump plus the schema.
