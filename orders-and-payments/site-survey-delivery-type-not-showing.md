# "Site Survey" never appears in `/delivery` — whitelisting a product cannot make it appear

QA whitelisted two products against the **Site Survey** delivery type (id 30) in the admin
(`SPEC-ACC-2020` / `662a289f0cc6d2313de0bdca`, then `LOC-ACC-0135` / `6835885f1fa26026c72285c5`)
and the type still does not show in the mobile app's "Available delivery options". The question was
why, and whether "added" (= whitelisted and not blacklisted) is the right mental model.

**Data provenance:** all figures verified read-only against the **dev database**
(`website_test` @ `54.171.181.199`) on **2026-09-10**, plus one read-only `POST` to the **dev API**
(`https://api.dev.firstfence.co.uk/delivery`) and a lookup in **dev Mongo**
(`cdn-test` @ `ec2-34-252-129-247`). Code read at `website-api` **`78ff46d`** (branch `mobile-app`,
2026-09-09), `ff-uk-mobile` `0566c88`, `gatsby-website` `22fe4a6ca`, `admin-website-v2` `c4287e6`.
No writes of any kind were made, and production was not touched.

Companion docs: [`delivery-types-and-options.md`](./delivery-types-and-options.md) (§10 corrects
two statements in it) and
[`duplicate-offload-options-investigation.md`](./duplicate-offload-options-investigation.md).

---

## TL;DR

**Site Survey has no row in `delivery_prices`, and `/delivery` builds its list by iterating
`delivery_prices` — not `delivery_types`. A delivery type with no price row is invisible at
checkout no matter what you whitelist.** QA's whitelist rows are correct and were written
correctly; they are simply operating on a type that never enters the loop.

1. **The whitelist data is fine.** Both rows exist in `delivery_whitelists` with the right Mongo
   ids, and both resolve to exactly the SKUs QA named (§3). Neither product is blacklisted against
   type 30. Nothing is wrong with what QA did in the admin.

2. **The mental model is the thing that's off.** A whitelist is a **restriction**, never an
   **enablement**. `enforce_whitelist` narrows an *already-available* delivery type down to
   specific products. It cannot add a delivery type to the response. Site Survey was never
   available to narrow (§1).

3. **`delivery_prices` is the real gate.** `getDeliveryAndCollectionTypes` iterates
   `closestWarehouse.deliveryPrices` and reads the delivery type *off each price row*
   (`deliveryTypes.js:39-40`). No price row → the type is never visited → `enabled`,
   `enforce_whitelist` and every whitelist row are all dead letters (§2).

4. **Site Survey is the only enabled delivery type with no price row.** 22 types exist; 19 are
   enabled; 18 have a full set of 7 price rows (one per depot). Type 30 has **0**. The other two
   unpriced types (21, 22) are both `enabled = 0`. That is precisely why "other delivery types do
   show up" (§3).

5. **It was never priced — this is not a deletion.** `delivery_prices` allocates a contiguous block
   of 7 ids per delivery type; the blocks for types 27 (130-136) and 31 (137-143) are adjacent with
   no gap, so no block was ever allocated for 28/29/30 (§3.2).

6. **Proved causally, not just correlationally.** Running the real
   `getDeliveryAndCollectionTypes` against the real dev rows, with the whitelist row present in
   both runs and *only* the price row differing, flips Site Survey from absent to present (§4).

7. **The screenshot is a different basket.** Neither whitelisted product has been in a dev cart
   since **2024-12-19** / **2025-07-16**. And `SPEC-ACC-2020` is blacklisted against Next Day
   (id 5) since 2024-10-07, so a basket containing it could not have rendered the "Next Day
   Delivery +£192.00" card in the screenshot (§6).

8. **Separately, that section shows one option by design.** `CheckoutScreen.tsx:307-320` renders a
   single card for the *selected* service; the full list is behind the tap-through bottom sheet.
   That is `ff-uk-mobile`'s call, not yours (§7).

9. **The fix is admin data, not code** — 7 `delivery_prices` rows, added per depot. But it has real
   traps: `sort_order = 0` would push Site Survey to the top of the list and make it the **default
   selected option** at £0.00 (§8). Do not do it without a product decision.

---

## 1. The literal question: does "added to a delivery type" mean it will show?

**No — and this is the crux.** QA's definition ("whitelisted and not blacklisted") is an accurate
description of the two tables, but it describes a *filter*, not a *switch*.

| What you might expect | What the code does |
|---|---|
| Whitelisting a product **adds** the delivery type for that product | Whitelisting **removes** the delivery type for every *other* product |
| A type with a whitelist entry becomes available | A type must **already** be available; the whitelist then narrows it |
| Not blacklisted ⇒ offered | Not blacklisted is necessary, not sufficient |

The availability of a delivery type is decided in this order, and the whitelist is step 5 of 5:

| # | Gate | Source | Site Survey (id 30) |
|---|---|---|---|
| 1 | Has a `delivery_prices` row **for the closest depot** | `delivery_prices` | ❌ **0 rows, any depot** |
| 2 | Not postcode-blocked | `delivery_post_codes` | ✅ 0 rules → never blocked |
| 3 | Not product-blacklisted | `delivery_blacklists` | ✅ no row for either product |
| 4 | `enabled = 1` | `delivery_types` | ✅ |
| 5 | Whitelist satisfied | `delivery_whitelists` | ✅ both products present |

Gate 1 fails, so gates 2-5 are never evaluated. QA got 4 of the 5 gates right; the one that was
missed is the one with no UI on that page (§8.1).

One further nuance worth passing to QA, because it will bite on the next test even after a price
row exists: `enforce_whitelist` requires **every** product in the basket to be whitelisted, not
just one (`deliveryTypes.js:100-110` — the loop sets `pushDelivery = false` on the *first*
non-whitelisted product). A basket of `SPEC-ACC-2020` + any third product will still not offer
Site Survey.

---

## 2. The mechanism — `/delivery` enumerates prices, not types

`POST /delivery` → `DeliveryController.index` → `initialize`
(`start/routes/public.js:64`, no route prefix, so the URL QA quoted is right).

`app/Controllers/Http/DeliveryController.js:399-408` loads depots and hangs everything off the
price rows:

```js
const depots = await Depot.query()
  .with("deliveryPrices")
  .with("deliveryPrices.depot")
  .with("deliveryPrices.deliveryType")
  .with("deliveryPrices.deliveryType.deliveryWhitelists")
  …
```

`DeliveryController.js:470-472` then passes **only the closest depot's** price rows:

```js
const allTypes = getDeliveryAndCollectionTypes({
  productIds,
  deliveryPrices: closestWarehouse.deliveryPrices,
```

and `app/Services/delivery/deliveryTypes.js:39-40` reads the type *out of* each price row:

```js
deliveryPrices.forEach((deliveryPrice) => {
  const { deliveryType } = deliveryPrice;
```

There is no other loop over `delivery_types` anywhere in the request. **`delivery_types` is only
ever reached through `delivery_prices`.** So the eager-loaded `deliveryWhitelists` on line 403 —
the very rows QA created — are only ever inspected for types that already have a price row.

Two consequences worth stating plainly:

- A delivery type with **no** price rows is invisible everywhere, for everyone.
- A delivery type priced at **some** depots is visible only to customers whose nearest depot is one
  of those. Partial pricing produces a location-dependent bug that is very hard to reproduce.

---

## 3. The data

### 3.1 Every delivery type vs. its price rows

`website_test`, 2026-09-10. `depots_priced` counts distinct depots with a `delivery_prices` row.

| id | name | type | enabled | collection | enforce_wl | **depots_priced** | wl rows |
|---|---|---|---|---|---|---|---|
| 1 | Standard Delivery | `STANDARD` | 1 | 0 | 0 | **7** | 0 |
| 5 | Next Day Delivery | `NEXT_DAY` | 1 | 0 | 0 | **7** | 0 |
| 7 | Bristol Depot | `BRISTOL` | 1 | 1 | 0 | **7** | 0 |
| 8 | Glasgow Depot | `GLASGOW` | 1 | 1 | 0 | **7** | 0 |
| 9 | Canvey Depot | `CANVEY` | 1 | 1 | 0 | **7** | 0 |
| 10 | Specific Day Delivery | `SPECIFIC` | 1 | 0 | 0 | **7** | 0 |
| 11 | Same Day UK Delivery | `SAME_DAY` | 1 | 0 | 0 | **7** | 0 |
| 12 | Express Delivery | `EXPRESS` | 1 | 0 | 0 | **7** | 0 |
| 14 | Tipton Depot | `TIPTON` | 1 | 1 | 0 | **7** | 0 |
| 17 | Economy Delivery | `ECONOMY` | 1 | 0 | 0 | **7** | 0 |
| 18 | Nottingham Depot | `NOTTINGHAM` | 1 | 1 | 0 | **7** | 0 |
| 21 | Standard Delivery January Offer! | `STANDARD_JANUARY_OFFER` | **0** | 0 | 0 | **0** | 0 |
| 22 | Express Delivery January Offer! | `EXPRESS_JANUARY_OFFER` | **0** | 0 | 0 | **0** | 0 |
| 23 | Free Delivery Direct to Site | `FREE_SPECIALISTGATES` | **0** | 0 | 1 | **0** | 1 |
| 25 | Head Office, Swadlincote | `HEAD_OFFICE` | 1 | 1 | 1 | **7** | **0** ⚠️ |
| 26 | Free Standard Delivery | `STANDARD_FREE` | 1 | 0 | 1 | **7** | 48 |
| 27 | DPD Shipping | `DPD` | 1 | 0 | 1 | **7** | 474 |
| **30** | **Site Survey** | **`SURVEY`** | **1** | 0 | **1** | **0** ⛔ | **2** |
| 31 | Courier Shipping | `COURIER_SHIPPING` | 1 | 0 | 1 | **7** | 1 |
| 32 | Heavy Goods Delivery | `SPECIALIST_GATES` | 1 | 0 | 1 | **7** | 56 |
| 33 | External Courier | `DIRECT_SUPPLIER` | 1 | 0 | 1 | **7** | 349 |
| 34 | Turnstile Delivery | `TURNSTILE` | 1 | 0 | 1 | **7** | 4 |

**22 types · 19 enabled · 18 priced · Site Survey is the only enabled type with zero price rows.**
126 price rows total = 18 types × 7 depots, exactly.

The full type-30 row, for the record:

```
id: 30 | name: Site Survey | type: SURVEY | subtitle: NULL | postcode: NULL | details: NULL
days: NULL | sort_order: 0 | price: NULL | collection: 0 | enabled: 1 | is_deleted: 0
enforce_whitelist: 1 | weight_limit: NULL | max_weight: NULL
created_at: 2024-05-30 09:46:42 | updated_at: 2026-09-10 06:42:37
```

`updated_at` is today — consistent with QA opening and saving the type this morning. Note
`type` is now `SURVEY`; the 2026-08-20 doc recorded it as `NULL` (§10).

### 3.2 Dating it: never priced, not deleted

`delivery_prices` allocates one contiguous block of 7 ids (one per depot) each time a type is
priced. The blocks in creation order:

| Block | delivery_type_id | id range |
|---|---|---|
| … | 17 | 87-93 |
| … | 18 | 94-100 |
| *(gap: 101-115 — 15 ids)* | — | — |
| … | 25 | 116-122 |
| … | 26 | 123-129 |
| … | 27 | **130-136** |
| … | 31 | **137-143** |
| … | 32 | 144-150 |
| … | 33 | 151-157 |
| … | 34 | 158-164 |

**Types 27 and 31 are adjacent with no gap between them.** If a block had ever been allocated for
28, 29 or 30 and later deleted, there would be a 7-, 14- or 21-id hole at 137. There is none.
`AUTO_INCREMENT` is 165, and all 126 surviving rows have `is_deleted = 0`.

**Conclusion: Site Survey has never had a price row since it was created on 2024-05-30.** This is
not a regression, nothing broke, and nothing deleted it. It has been non-functional for
**~15 months**. The 15-id gap at 101-115 is old and belongs to types 21/22/23 (the disabled January
offers); it is not related.

### 3.3 The whitelist rows QA created are correct

```
id   | product_id                | delivery_type_id | created_at
1194 | 662a289f0cc6d2313de0bdca  | 30               | 2026-09-10 06:36:30
1195 | 6835885f1fa26026c72285c5  | 30               | 2026-09-10 11:46:01
```

Resolved against dev Mongo `cdn-test.products`:

| `product_id` | `sku` | `title` | `status` |
|---|---|---|---|
| `662a289f0cc6d2313de0bdca` | `SPEC-ACC-2020` | BOB50ME Above Ground Swing Gate Operator | PUBLISHED |
| `6835885f1fa26026c72285c5` | `LOC-ACC-0135` | Locinox B-Safe Stainless Steel Safety Cable | PUBLISHED |

Exactly the two products QA named. Correct ids, correct type, no casing problem, no duplicates.
`delivery_whitelists.product_id` is `VARCHAR(500)`
(`database/migrations/1667481534964_delivery_whitelist_schema.js:10`) and the admin writes the
Mongo `_id` hex string
(`admin-website-v2/src/pages/content/delivery/delivery-types/delivery-type-page/whitelisted-products/whitelisted-products.jsx:56-62`),
which is what `DeliveryController.js:412` matches on (`products.map((product) => product.mongo_id)`).
**The write path is sound.**

---

## 4. Proof that the price row is the cause

One matching hypothesis is not a conclusion, so I ran the real service function with the real dev
rows, changing exactly one thing. The whitelist row, `enforce_whitelist: 1`, `enabled: 1` and the
`productIds` array are **identical in both runs**; the only difference is whether a
`delivery_prices` row for type 30 is in the input array.

```
A) dev reality: NO delivery_prices row for type 30  -> [{"id":1,"name":"Standard Delivery","price":45.39}]
B) same data + one delivery_prices row for type 30  -> [{"id":30,"name":"Site Survey","price":0},
                                                       {"id":1,"name":"Standard Delivery","price":45.39}]
```

(Harness in Appendix B; it `require`s `app/Services/delivery/deliveryTypes.js` unmodified and stubs
only the Adonis `use("Helpers/MathHelper")` global.)

Note run B also demonstrates the §8.2 trap: Site Survey sorts to the **front** of the list at
**£0.00**.

---

## 5. Ruling out the alternatives

| Hypothesis | Ruled out by |
|---|---|
| Whitelist row wrong / wrong id format / casing | Both rows resolve to the named SKUs in Mongo (§3.3); `product_id` is `VARCHAR(500)`, ids are exact 24-hex |
| Product blacklisted against type 30 | No `delivery_blacklists` row pairs either product with type 30 (§6 table) |
| Type disabled or soft-deleted | `enabled = 1`, `is_deleted = 0` (§3.1) |
| Postcode rules exclude it | 0 rows in `delivery_post_codes` for type 30 → the `deliveryPostCodes.length > 0` branch (`deliveryTypes.js:54-57`) never runs |
| Blocked dates | 0 rows in `delivery_blocked_dates` for type 30 |
| Weight limit filter | `weight_limit` and `max_weight` are both `NULL` → `DeliveryController.js:541-543` returns `true` early |
| A special-product branch replaces the list (gate operator → residential gate?) | Neither `SPEC-ACC-2020` nor `LOC-ACC-0135` appears anywhere in `app/`, `start/` or `database/`. All six branch predicates (`containsResidentialGateProduct`, concrete, cantilever, turnstile, EnviroRail, steel road plate) test `product.sku` against hard-coded lists in `app/Utils/delivery-constants.js`. The `else` branch runs. |
| Installation filter | `filterDeliveryTypesInstallation` only narrows when a product carries installation; neither does |
| Mobile client filters the array | `ff-uk-mobile/src/features/checkout/services/deliveryApi.ts:123` is a bare `.map` — no `.filter`, `.slice`, cap or allow-list anywhere in the checkout feature |
| Mobile cart identity broken (cookie vs header) | `resolveCartIdentity` (`app/Utils/cartIdentity.js:42-48`) takes `X-Cart-Token` with **absolute precedence**; the mobile client sets it (`restApi.ts:20`). Working as designed. |
| Stale RTK Query cache (60 s, keyed on postcode) | Plausible for a *transient* miss, but cannot explain a persistent one — and is moot given the type cannot be returned at all |
| `is_deleted` hiding price rows | All 126 `delivery_prices` rows have `is_deleted = 0`; the eager load does not filter it anyway |

The dev API itself, called read-only with an existing dev cart, returns four types and no Site
Survey — consistent with all of the above:

```
POST https://api.dev.firstfence.co.uk/delivery   {"postcode":"DE11 8LQ"}   → HTTP 200
delivery: [ECONOMY 17 £80, STANDARD 1 £87.50, EXPRESS 12 £109.50, SPECIFIC 10 £0]
```

---

## 6. The screenshot is a different basket

Worth separating from the main finding, because it means the screenshot cannot be used as evidence
about these two products either way.

**Neither product has been in a dev cart in 2026:**

| `mongo_id` | sku | times in a cart | first | last |
|---|---|---|---|---|
| `662a289f0cc6d2313de0bdca` | SPEC-ACC-2020 | 10 | 2024-06-19 | **2024-12-19** |
| `6835885f1fa26026c72285c5` | LOC-ACC-0135 | 6 | 2025-07-16 | **2025-07-16** |

Carts are being created on dev normally (e.g. cart 4869508 at 2026-09-10 10:14:30 with 3 SKUs), so
this is not a gap in the data.

**And the screenshot contradicts the basket containing `SPEC-ACC-2020`:** that product carries 19
`delivery_blacklists` rows created 2024-10-07, including **`delivery_type_id = 5`, Next Day
Delivery**. With it in the basket, Next Day is pushed onto `blacklist` and excluded at
`deliveryTypes.js:113`. The screenshot shows "Next Day Delivery +£192.00", so the basket did not
contain it. (For that product the only non-blacklisted priced types are **1 Standard** and
**33 External Courier**, which is itself worth a QA note.)

This does not change the conclusion — type 30 cannot appear for *any* basket — but QA should retest
with the product actually in the basket before drawing further inferences.

---

## 7. The mobile UI shows one option by design — `ff-uk-mobile`, not yours

Flagging plainly: **this part is not a `website-api` fix.**

`ff-uk-mobile/src/features/checkout/screens/CheckoutScreen.tsx:307-320` does not render a list under
"Available delivery options". It renders a single `ExpandableOptionCard` for the currently selected
service, and the selection defaults to the first array element
(`useDeliverySelection.ts:54-56`, `return list.find(...) ?? list[0] ?? null`). The full list only
appears after tapping the card, in the `OptionPickerSheet` — which reuses the same
"Available delivery options" title (`useDeliverySelection.ts:161`).

`gatsby-website` renders all options inline as radio cards
(`src/features/checkout/service-page/delivery-options/delivery-options.js:29,46`), which is very
likely where the expectation "the section lists the available options" comes from. The two clients
present the same payload differently.

So "only Next Day shows" is the expected rendering of that section regardless of how many options
the API returns. **Even so, it is not the explanation here** — Site Survey is absent from the
payload, not merely collapsed behind the card. Both statements are true at once; only the backend
one is causal.

Also `ff-uk-mobile/docs/ARCHITECTURE.md:187` still marks Checkout as **mock** (`cart.mock`). That
file no longer exists in the repo and checkout is on the real REST API
(`ARCHITECTURE.md:20`). That doc row is stale and should not be trusted when triaging.

---

## 8. If you want Site Survey to actually work

### 8.1 Why it was missed — the admin has no Prices tab on this page

The delivery-type page has exactly five tabs — General, Day Options, Postcodes, Blacklisted
Products, Whitelisted Products
(`admin-website-v2/.../delivery-type-page/delivery-type-page.jsx:129-159`). **There is no Prices
tab, and the page never displays whether the type is priced.** Delivery prices are managed from the
*other* end of the relationship: **Content → Delivery → Depots → \<depot\> → Delivery Prices → Add**
(`admin-website-v2/src/pages/content/delivery/depots/depot-page/delivery-prices/`), where the modal
asks for Min Price, Price Per Mile and a delivery-type dropdown
(`delivery-prices-modal.jsx:24-45`).

And `DeliveryTypeController.store` (`app/Controllers/Http/DeliveryTypeController.js:11-46`) creates
**only** the `delivery_types` row — no price rows. The single creation path for `delivery_prices` in
the whole API is `DeliveryPriceController.js:28`, driven by that depot modal.

**So every newly created delivery type is silently non-functional until someone visits all 7 depot
pages.** Nothing warns you. That is the systemic trap here, and it will recur.

### 8.2 Traps before adding the rows

| Trap | Detail |
|---|---|
| **Needs all 7 depots** | Only `closestWarehouse.deliveryPrices` is used (`DeliveryController.js:471`). Pricing one depot makes the option appear only for customers nearest that depot — a location-dependent bug. Depots: 1 Swadlincote, 11 Glasgow, 13 Canvey, 14 Bristol, 15 Tipton, 16 Bradford, 19 Nottingham. |
| **`sort_order = 0` → becomes the default** | Sorted ascending (`deliveryTypes.js:146`), 0 ties with Economy Delivery. Both clients preselect `delivery[0]` (`useDeliverySelection.ts:85`; `gatsby .../checkout/delivery/index.js:147-149`). In the §4 harness it landed **first**. Set a sensible `sort_order` in the same change. |
| **£0.00 and no lead time** | `price_per_mile`/`min_price` of 0 render as "+£0.00"; `days` is `NULL` and there are no `delivery_day_options`, so the app falls back to `deliveryDates[0]`. A survey is presumably not a priced delivery at all — worth a product decision, not a data patch. |
| **`enforce_whitelist` is all-or-nothing** | Every basket product must be whitelisted (§1). With 2 products whitelisted, only baskets composed solely of those 2 qualify. |
| **No `SURVEY` handling exists anywhere** | `type = 'SURVEY'` is referenced by **no** code in `website-api`, `gatsby-website`, `ff-uk-mobile` or `admin-website-v2`. (The admin's "Site Surveys" pages are an unrelated feature — `site_surveys`, survey emails — not this delivery type.) It would flow through as a generic priced delivery option and be stored on the order as one. |

### 8.3 Reversibility

Adding price rows is additive and reversible — `DELETE FROM delivery_prices WHERE
delivery_type_id = 30` restores today's behaviour exactly, since nothing else references those rows.

The FK delete rules, if the alternative is to retire type 30 instead:

| Referencing table | ON DELETE |
|---|---|
| `deliveries` | **SET NULL** ⚠️ historical rows lose their type |
| `delivery_date_type_pivot` | **CASCADE** |
| `delivery_prices`, `delivery_whitelists`, `delivery_blacklists`, `delivery_day_options`, `delivery_post_codes` | NO ACTION (the delete is blocked while rows exist) |

**Do not hard-delete delivery type 30.** `deliveries.delivery_type_fk` is `SET NULL`, so it would
silently blank the type on any historical delivery row pointing at it. Set `enabled = 0` instead —
which costs nothing today, as it is already unreachable.

**My recommendation:** don't add the price rows yet. The type has been dead for 15 months, nothing
consumes `SURVEY`, and pricing it makes a £0.00 option the checkout default. Get the intended
behaviour defined first; the one-line safe change now is `enabled = 0` so it stops looking
configurable in the admin.

---

## 9. "Is this just dev?" — three separate questions

**Q1 — Can a deploy or pipeline cause this on prod?**
**No.** `website-api/.gitlab-ci.yml` deploys with `git fetch` + `git reset --hard origin/master` +
`pm2 restart 0`, gated on `only: - master`. It runs **no** `adonis migration:run` and **no**
`adonis seed`. No seeder in `database/seeds/` touches `delivery_prices` or `delivery_types` at all
(`DeliveryOffloadSeeder`, `DeliveryParamSeeder`, `DepotSeeder` and the rest — grep for
`delivery_prices`/`delivery_types` across `database/seeds/` returns nothing). These rows are
admin-entered data only.

**Q2 — Can it happen again after the fixes already in the repo?**
**Yes — there is no fix in the repo, and nothing prevents it.** Creating a delivery type writes one
row and no prices (§8.1); no code, validator or UI checks that a type is priced; the delivery-type
page cannot even display prices. Any delivery type created from now on starts life invisible. This
is a standing trap, not a past incident.

**Q3 — Is prod already in that state?**
**Almost certainly yes, but I did not verify it — prod was not touched.** Type 30 was created
`2024-05-30 09:46:42`, well before the dev database was restored (`information_schema` `CREATE_TIME`
for every one of these tables is `2026-06-24 12:28:1x`, i.e. the dump import), so the row and its
`created_at` came from the source system rather than being made on dev. Since prod is where it
originated and no pipeline step could have removed the price rows, prod is expected to match.

Exact read-only query to settle it yourself, against **prod** MySQL:

```sql
-- Any enabled delivery type that no depot prices = invisible at checkout.
SELECT dt.id, dt.name, dt.type, dt.enabled, dt.enforce_whitelist,
       COUNT(DISTINCT dp.depot_id) AS depots_priced,
       (SELECT COUNT(*) FROM delivery_whitelists w WHERE w.delivery_type_id = dt.id) AS wl_rows
FROM   delivery_types dt
LEFT   JOIN delivery_prices dp ON dp.delivery_type_id = dt.id
GROUP  BY dt.id
HAVING depots_priced < (SELECT COUNT(DISTINCT depot_id) FROM delivery_prices)
ORDER  BY dt.enabled DESC, dt.id;
```

Any row with `enabled = 1` is a delivery type the admin presents as configurable that no customer
can ever be offered.

---

## 10. Reconciliation with existing docs

**[`delivery-types-and-options.md`](./delivery-types-and-options.md) (2026-08-20) — two corrections
needed.**

1. **§3 opening is wrong and it is the reason this bug is invisible.** It says
   `getDeliveryAndCollectionTypes` *"iterates every row and pushes it into one of two arrays, then
   filters by `enabled`."* It iterates **`deliveryPrices` for the closest depot** and reaches the
   type through `deliveryPrice.deliveryType` (`deliveryTypes.js:39-40`). A `delivery_types` row with
   no price row is never iterated. Reading that sentence, "id 30, enabled ✅" in §3.1 looks like a
   live option — it is not, and never has been. *(Was wrong when written; the measurement wins.)*

2. **§3.1 row for id 30 is now stale.** It records Site Survey with `type` **`NULL`** and no
   whitelist. Today `type = 'SURVEY'` and there are 2 whitelist rows — both changed on 2026-09-10
   by QA (`delivery_types.updated_at 06:42:37`; whitelist rows 06:36:30 and 11:46:01). *(Correct
   when written.)*

Everything else in that doc that I crossed checks out. In particular §7.2's rate table lists
12 delivery + 6 collection = **18** types with rates against the 22 in §3.1 — the doc already
contained the evidence for this bug (types 21, 22, 23 and 30 have no rate) without drawing the
conclusion. Adding a `depots_priced` column to §3.1 would make it self-evident.

**Nothing else covers this.** Greps across `website-api/.generated_docs/` and
`cdn-graphql-v2/.generated_docs/` for `whitelist`/`blacklist` return **zero** matches; the
"site survey" hits there are all the unrelated dead `send::site-survey` order event
(e.g. `epic-3-checkout/FIR-18-Mobile-Payments-Backend-Strategy.md:28`). No FIR ticket covers
delivery-type whitelisting.

**`ff-uk-mobile/docs/ARCHITECTURE.md:187`** is stale — see §7.

---

## 11. Adjacent findings (not the cause, but in the same code)

Listed because they are one-line reads from the same tables and each is a live footgun.

1. **`enforce_whitelist` with an empty whitelist means *no* restriction, not *total* restriction.**
   `deliveryTypes.js:128-130` short-circuits on `deliveryType.deliveryWhitelists.length === 0`.
   **Head Office, Swadlincote (id 25)** is `enforce_whitelist = 1` with **0** whitelist rows and 7
   price rows — so it is offered to **everyone**, which is the exact opposite of what the flag
   reads like in the admin. It is the only type in this state today.

2. **`DeliveryTypeController.update:91-99` has an always-true condition.**
   `enforceWhitelist !== null || enforceWhitelist !== undefined` can never be false, so
   `enforce_whitelist` (and `enabled`, same shape) is **always** overwritten with
   `Boolean(payload.…)` — any partial update that omits the field silently resets it to `false`.
   The admin masks this by POSTing the whole object; any other caller would not.

3. **`containsOnlyEnviroRailProduct` returns `true` for an empty basket.**
   `envirorail-products.js:5-12` starts `true` and only falsifies inside `forEach`, and
   `ENVIRORAIL_PRODUCTS` is `[]` (`delivery-constants.js:129`). An empty `products` array would
   route into the EnviroRail branch and return an empty delivery list. The `cartProducts`
   middleware 400s on an empty cart (`CartProduct.js:49-58`) so it is currently unreachable — worth
   knowing before that guard is relaxed.

4. **`delivery_blacklists.is_deleted` and `delivery_prices.is_deleted` are dead columns.** All
   29,441 and 126 rows respectively are `0`, and no read path filters on either. "Removing" a
   blacklist entry in the admin is a hard delete.

5. **`DeliveryWhitelistController.js:114` calls `ObjectID(item.product_id)` unguarded** — one
   malformed `product_id` row 500s the whole Whitelisted Products tab. The blacklist equivalent
   (`DeliveryBlacklistController.js:160-171`) already try/catches this. There is no validator on any
   whitelist/blacklist route, so nothing stops a bad value being stored.

---

## 12. What I did NOT verify

- **Production, at all.** Q3 in §9 is an inference from `created_at` and the pipeline contents, not
  a measurement. The query to settle it is in §9.
- **Which API the QA build pointed at.** `ff-uk-mobile/.env.local:8` sets a **local**
  `EXPO_PUBLIC_API_BASE_URL` with the dev URL commented out on line 11. If the build hit a local
  `website-api`, the dev whitelist rows were never in play. It does not change the conclusion
  (type 30 has no price row on dev either), but it would change what the screenshot means.
- **The actual basket in the screenshot.** I showed it contained neither whitelisted product (§6); I
  did not identify which cart it was.
- **Whether a Site Survey price row ever existed on *prod*.** The id-block argument in §3.2 is dev
  data only.
- **Anything about how Site Survey is *supposed* to behave.** No spec, ticket or code exists for
  `type = 'SURVEY'`; §8 describes what would happen mechanically, not what is intended.
- **`enforce_whitelist`'s presence on the prod schema.** Migration history for `delivery_types` was
  rewritten (`71d27d1`, 2026-01-21, "Fix migrations" — the create-table file was renamed, so Adonis'
  filename-keyed ledger orphans the old row). I confirmed the column exists on **dev** by reading
  the row; the migration files are not reliable evidence for any other environment.
- **Load/perf.** `deliveryPrices.deliveryType.deliveryWhitelists` eager-loads *all* whitelist rows
  per type on every `/delivery` call — 474 for DPD, 349 for External Courier, 932 rows total. I did
  not measure the cost.

---

## Appendix A — queries used

All read-only against `website_test` @ `54.171.181.199` on 2026-09-10, via
`docker exec firstfence-mysql mysql -h 54.171.181.199 -u mobile_dev -p… website_test`.

```sql
-- §3.1 the whole picture
SELECT dt.id, dt.name, dt.type, dt.enabled, dt.is_deleted, dt.collection,
       dt.enforce_whitelist, dt.sort_order,
       COUNT(DISTINCT dp.depot_id) AS depots_priced,
       (SELECT COUNT(*) FROM delivery_whitelists w WHERE w.delivery_type_id = dt.id) AS wl_rows
FROM delivery_types dt
LEFT JOIN delivery_prices dp ON dp.delivery_type_id = dt.id
GROUP BY dt.id ORDER BY dt.id;

-- the type row itself
SELECT * FROM delivery_types WHERE id = 30\G
SELECT * FROM delivery_prices WHERE delivery_type_id = 30;      -- 0 rows
SELECT * FROM delivery_whitelists WHERE delivery_type_id = 30;  -- 2 rows

-- §3.2 id-block allocation: proves 30 was never priced
SELECT delivery_type_id, COUNT(*) n, MIN(id) min_id, MAX(id) max_id,
       GROUP_CONCAT(depot_id ORDER BY depot_id) depots
FROM delivery_prices GROUP BY delivery_type_id ORDER BY MIN(id);

SELECT TABLE_NAME, AUTO_INCREMENT, CREATE_TIME, UPDATE_TIME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'website_test'
  AND TABLE_NAME IN ('delivery_prices','delivery_types','delivery_whitelists',
                     'delivery_blacklists','delivery_day_options','delivery_post_codes');

-- §5 the other gates
SELECT b.*, dt.name FROM delivery_blacklists b
  JOIN delivery_types dt ON dt.id = b.delivery_type_id
  WHERE b.product_id IN ('662a289f0cc6d2313de0bdca','6835885f1fa26026c72285c5');
SELECT * FROM delivery_day_options   WHERE delivery_type_id = 30;  -- 0 rows
SELECT COUNT(*) FROM delivery_post_codes WHERE delivery_type_id = 30;  -- 0
SELECT * FROM delivery_blocked_dates WHERE delivery_type_id = 30;  -- 0 rows
SELECT is_deleted, COUNT(*) FROM delivery_prices     GROUP BY is_deleted;  -- all 0
SELECT is_deleted, COUNT(*) FROM delivery_blacklists GROUP BY is_deleted;  -- all 0

-- §6 basket forensics
SELECT mongo_id, COUNT(*) n, MIN(created_at) first_seen, MAX(created_at) last_seen
FROM products WHERE mongo_id IN ('662a289f0cc6d2313de0bdca','6835885f1fa26026c72285c5')
GROUP BY mongo_id;

SELECT c.id, c.completed, c.created_at, COUNT(p.id) n_products,
       GROUP_CONCAT(DISTINCT p.sku ORDER BY p.sku SEPARATOR ' | ') skus
FROM carts c LEFT JOIN products p ON p.cart_ID = c.id
WHERE c.updated_at >= '2026-09-08' GROUP BY c.id ORDER BY c.updated_at DESC LIMIT 20;

-- §8.3 FK delete rules
SELECT CONSTRAINT_NAME, TABLE_NAME, UPDATE_RULE, DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'website_test' AND REFERENCED_TABLE_NAME = 'delivery_types';

-- §11.1 the empty-whitelist footgun
SELECT dt.id, dt.name, dt.enabled, dt.collection, COUNT(w.id) wl
FROM delivery_types dt LEFT JOIN delivery_whitelists w ON w.delivery_type_id = dt.id
WHERE dt.enforce_whitelist = 1 GROUP BY dt.id HAVING wl = 0;
```

Dev Mongo (`cdn-test` @ `ec2-34-252-129-247`, `authSource=cdn-test`):

```js
["662a289f0cc6d2313de0bdca","6835885f1fa26026c72285c5"].forEach(s => {
  const d = db.products.findOne({ _id: ObjectId(s) });
  printjson({ _id: String(d._id), sku: d.sku, title: d.title, status: d.status, weight: d.weight });
});
```

Dev API (read-only; `POST /delivery` performs no writes — it reads the cart named by the header and
computes):

```bash
curl -s -X POST 'https://api.dev.firstfence.co.uk/delivery' \
  -H 'Content-Type: application/json' \
  -H 'X-Cart-Token: <sessionID of an existing dev cart>' \
  -H 'x-ff-app: 1' \
  -d '{"postcode":"DE11 8LQ"}' | jq '[.delivery[] | {id,type,name,price}]'
```

## Appendix B — the §4 causal harness

Requires the real service module unmodified; stubs only the Adonis `use()` global.

```js
global.use = (name) => {
  if (name === "Helpers/MathHelper") {
    return { getDistanceFromLatLonInMiles: () => 25, round: (n) => n,
             sortByDistance: (arr) => arr, haversineDistance: () => 25 };
  }
  throw new Error("unexpected use(): " + name);
};
const { getDeliveryAndCollectionTypes } =
  require("<repo>/website-api/app/Services/delivery/deliveryTypes.js");

const siteSurvey = () => ({                      // real dev row, delivery_types id=30
  id: 30, name: "Site Survey", type: "SURVEY", sort_order: 0, price: null,
  collection: 0, enabled: 1, is_deleted: 0, enforce_whitelist: 1,
  weight_limit: null, max_weight: null,
  deliveryWhitelists: [{ product_id: "662a289f0cc6d2313de0bdca", delivery_type_id: 30 }],
  deliveryPostCodes: [], deliveryDayOptions: [], deliveryBlockedDates: [],
});
const standard = () => ({                        // control, delivery_types id=1
  id: 1, name: "Standard Delivery", type: "STANDARD", enabled: 1, collection: 0,
  enforce_whitelist: 0, sort_order: 1, deliveryWhitelists: [],
  deliveryPostCodes: [], deliveryDayOptions: [], deliveryBlockedDates: [],
});

const common = {
  lowestDistance: 25, maxDeliveryMultiplier: 1, postcode: "DE118LQ",
  allDeliverySurcharge: 1, deliveryLocation: { lat: 52.77, lng: -1.55 },
  blacklist: [], whitelist: [30], productIds: ["662a289f0cc6d2313de0bdca"],
};

const priceStandard = { price_per_mile: 0.90, min_price: 45.39, deliveryType: standard() };
const priceSurvey   = { price_per_mile: 0.00, min_price: 0.00,  deliveryType: siteSurvey() };

const run = (label, deliveryPrices) =>
  console.log(label, "->", JSON.stringify(
    getDeliveryAndCollectionTypes({ ...common, deliveryPrices })
      .deliveryTypes.map(d => ({ id: d.id, name: d.name, price: d.price }))));

run("A) NO price row for type 30 ", [priceStandard]);
run("B) WITH a price row for 30  ", [priceStandard, priceSurvey]);
```

---

## Code reference index

| Concern | Location |
|---|---|
| Route | `website-api/start/routes/public.js:64` |
| Products come from the cart, not the payload | `website-api/app/Middleware/CartProduct.js:60-61` |
| Cart identity (`X-Cart-Token` wins) | `website-api/app/Utils/cartIdentity.js:42-48` |
| Eager load hangs off price rows | `website-api/app/Controllers/Http/DeliveryController.js:399-408` |
| Closest depot's prices only | `website-api/app/Controllers/Http/DeliveryController.js:470-472` |
| **The loop that decides availability** | `website-api/app/Services/delivery/deliveryTypes.js:39-40` |
| `enforce_whitelist` = all products | `website-api/app/Services/delivery/deliveryTypes.js:100-110` |
| Whitelist short-circuit on empty list | `website-api/app/Services/delivery/deliveryTypes.js:128-130` |
| Blacklist exclusion | `website-api/app/Services/delivery/deliveryTypes.js:113` |
| `enabled` filter | `website-api/app/Services/delivery/deliveryTypes.js:149-151` |
| Type created without prices | `website-api/app/Controllers/Http/DeliveryTypeController.js:11-46` |
| Only price-creation path | `website-api/app/Controllers/Http/DeliveryPriceController.js:28` |
| Whitelist store (no validator) | `website-api/app/Controllers/Http/DeliveryWhitelistController.js:13-23` |
| Whitelist table schema | `website-api/database/migrations/1667481534964_delivery_whitelist_schema.js:10` |
| Admin: no Prices tab | `admin-website-v2/.../delivery-type-page/delivery-type-page.jsx:129-159` |
| Admin: where prices live | `admin-website-v2/src/pages/content/delivery/depots/depot-page/delivery-prices/` |
| Admin: whitelist add payload | `admin-website-v2/.../whitelisted-products/whitelisted-products.jsx:56-62` |
| Mobile: single-card render | `ff-uk-mobile/src/features/checkout/screens/CheckoutScreen.tsx:307-320` |
| Mobile: default = `list[0]` | `ff-uk-mobile/src/features/checkout/hooks/useDeliverySelection.ts:54-56` |
| Mobile: no client filtering | `ff-uk-mobile/src/features/checkout/services/deliveryApi.ts:123` |
| Gatsby: renders full list | `gatsby-website/src/features/checkout/service-page/delivery-options/delivery-options.js:29,46` |
| Deploy pipeline (no migrate/seed) | `website-api/.gitlab-ci.yml` |
