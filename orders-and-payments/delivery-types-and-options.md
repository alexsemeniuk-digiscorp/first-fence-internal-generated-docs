# Delivery — Every Type, Option and Value the System Can Produce

Reference for QA: what "delivery type" actually means in this system, the complete set of values
each field can hold, and — critically — the set of values actually **stored in the database**,
which is much larger and messier than the code's enums.

**Data provenance:** all figures verified read-only against the **dev database**
(`website_test` @ `54.171.181.199`) on 2026-08-20 — **61,727 `deliveries` rows**, 22 delivery
types, 4 offloads, 26 pricing variables, 3,537 postcode rules. Where live data differs from the
repo's seed files, the live value is given and the seed is noted.

Companion doc:
[`order-and-payment-status-flow-trade-credit.md`](./order-and-payment-status-flow-trade-credit.md)
(how delivery prices land on an order).

---

## TL;DR

**"Delivery type" is five different fields, and the database contains far more values than the code
knows about.** A single order's delivery is described by all five at once:

| # | Axis | Column | Values the **code** defines | Values actually **in the DB** |
|---|---|---|---|---|
| 1 | **Service** | `delivery_type` | 2 (`DELIVERY`, `COLLECTION`) | **7** |
| 2 | **Named service** | `delivery_type_fk` | 4 IDs hardcoded | **22 rows** (16 delivery + 6 collection) |
| 3 | **Time frame** | `delivery_time_frame` | 6 mapped to SAP SKUs | **36** |
| 4 | **Time slot** | `time_type` | 4 (`ALL_DAY`,`AM`,`PM`,`TIME_SLOT`) | **8** |
| 5 | **Offload** | `offload_type` | 3 in constants, 4 in DB | **8** |

Layered on top are **six end-to-end scenarios** that change how delivery behaves rather than just
what gets stored: sale delivery, depot collection, hire out-and-return, installation-bundled
(which leaves exactly **one** delivery option), concrete/bulk, and non-UK export. See §1.

**The headline for QA: none of these columns is an enum, none has a validator, and every
comparison in the code is case-sensitive `===`.** The result is quantifiable breakage in live data:

| Bug | Rows affected | Effect |
|---|---|---|
| Title-case `Delivery` / `Collection` fail the Salesforce quote gate | **7,273** (11.8%) | **no delivery line on the CPQ quote at all** |
| `Collection` (title case) fails `=== 'COLLECTION'` | **1,237** | order emails say "Delivery" for a collection |
| lowercase `assisted_offload` / `first_fence_offload` | **148** | chargeable offload (£43 / £108) never reaches the quote |
| `delivery_time_frame` values with no SAP SKU | **~2,777** | no delivery SKU on the quote — `DPD` alone is 1,628 |
| `Next Day` stored with a space, not `NEXT_DAY` | **738** | maps to nothing, despite `NEXT_DAY` being a valid key |
| `ANY_TIME` / `SLOT` / `CUSTOM_TIME` not in the enum | **1,527** | rendered as a bogus 2-hour window in emails |

Details and evidence in §8.

---

## 1. The six delivery scenarios

"Type" and "kind" mean the same thing here — there is no separate `kind` field. But the question
has two equally valid answers, and QA needs both: **the enum values** a record can hold (§2–§6),
and **the end-to-end scenarios** that change behaviour. The latter:

| Scenario | How it's identified | What changes |
|---|---|---|
| **1. Sale delivery to an address** | `delivery_type = DELIVERY` | the baseline |
| **2. Collection from a depot** | `delivery_type = COLLECTION` | `collection_location` set; one SAP SKU (`SERV-COL-0001`) with **price forced to 0**; time frame irrelevant |
| **3. Hire out-and-return** | any product with `productType = "HIRE"` | own vocabulary + doubled pricing — §1.1 |
| **4. Installation-bundled** | any basket line with installation | **all delivery types except `SPECIFIC` are removed** (`installation.js:8-18`). Live data confirms exactly **one** row qualifies (id 10, "Specific Day Delivery"), so selecting installation collapses 13 enabled options to 1 |
| **5. Concrete / bulk** | concrete products in basket | `DeliveryController` returns a **separate response branch** (`concreteDeliveryTypes` / `concreteDeliveryDates`); fleet sized by weight — rigid 10,160 kg, artic 26,417 kg (`vehicles.js`) |
| **6. Non-UK / export** | `orders.non_uk = 1` | set from the `nonUK` request flag (`OrderController.js:499,582`) |

A seventh variant exists at the feed level: types flagged `is_google_shopping` (live: only
`Economy Delivery`, id 17) are exposed to the Google Shopping feed.

### 1.1 Hire is the most distinct scenario

`app/Services/delivery/hire.js` treats a hire order as **two movements** and prices them together:

1. **Its own day-type vocabulary** — human-readable strings, not tokens (`hire.js:22-31`):
   `"Delivery"` · `"Collection"` · `"Delivery And Collection"`
2. **Same-day out-and-back costs double** — `sameDayModifier = 2` (`hire.js:41-46,59`)
3. **Collection depot prices doubled** unconditionally — `item.price *= 2` (`hire.js:65-69`)
4. **Offload multiplied by the number of unique hire dates** — `item.price *= numUniqueDates`
   (`hire.js:70-73`)

The hire period is persisted as JSON in `deliveries.hire_dates` (`LONGTEXT`). `getShippingService`
relabels `Collection` → **`Collect/Return`** and appends `"(Dates Above)"` / `"(All)"`
(`shipping-services.js:123,134,139,145,156`). `MIN_HIRE_TERM = 14` days.

Live data: 278 rows have `delivery_time_frame = 'HIRE'` and 8 have
`'HIRE DELIVERY AND COLLECTION'`.

---

## 2. Service — `delivery_type`

Code defines two (`constants.js:45-48`): `SERVICES = { DELIVERY, COLLECTION }`.

**The database holds seven:**

| Value | Rows | Note |
|---|---|---|
| `DELIVERY` | 45,147 | canonical |
| `COLLECTION` | 9,270 | canonical |
| `Delivery` | 6,041 | ⚠️ title case |
| `Collection` | 1,232 | ⚠️ title case |
| `NULL` | 24 | |
| `Economy Delivery` | 10 | ⚠️ a delivery-type **name** leaked into the service column |
| `Standard Delivery` | 3 | ⚠️ same |

`varchar(256)`, no enum, no validator. `getService()` (`shipping-services.js:10-15`) treats
**anything that isn't exactly `COLLECTION` as `Delivery`**.

---

## 3. Named service — the `delivery_types` table

Admin-managed (`admin.js:108-114`), split into delivery options vs collection depots by the
**`collection` boolean** — collection is *not* a separate table. `getDeliveryAndCollectionTypes`
(`app/Services/delivery/deliveryTypes.js`) iterates every row and pushes it into one of two arrays,
then filters by `enabled`.

### 3.1 Delivery options (live — 16 rows, 13 enabled)

| id | name | `type` | days | enabled | whitelist | `price` |
|---|---|---|---|---|---|---|
| 1 | Standard Delivery | `STANDARD` | 5 | ✅ | | 0.90 |
| 12 | Express Delivery | `EXPRESS` | 3 | ✅ | | 1.30 |
| 5 | Next Day Delivery | `NEXT_DAY` | 1 | ✅ | | 1.80 |
| 11 | Same Day UK Delivery | `SAME_DAY` | 0 | ✅ | | 180.00 |
| 10 | Specific Day Delivery | `SPECIFIC` | 0 | ✅ | | — |
| 17 | Economy Delivery | `ECONOMY` | 7 | ✅ | | — |
| 26 | Free Standard Delivery | `STANDARD_FREE` | — | ✅ | ✅ | — |
| 27 | DPD Shipping | `DPD` | 2 | ✅ | ✅ | — |
| 31 | Courier Shipping | `COURIER_SHIPPING` | 3 | ✅ | ✅ | — |
| 32 | Heavy Goods Delivery | `SPECIALIST_GATES` | 42 | ✅ | ✅ | — |
| 33 | External Courier | `DIRECT_SUPPLIER` | 2 | ✅ | ✅ | — |
| 34 | Turnstile Delivery | `TURNSTILE` | — | ✅ | ✅ | — |
| 30 | Site Survey | **`NULL`** | — | ✅ | | — |
| 21 | Standard Delivery January Offer! | `STANDARD_JANUARY_OFFER` | — | ❌ | | — |
| 22 | Express Delivery January Offer! | `EXPRESS_JANUARY_OFFER` | — | ❌ | | — |
| 23 | Free Delivery Direct to Site | `FREE_SPECIALISTGATES` | — | ❌ | ✅ | — |

Note the `price` column is a **multiplier** for ids 1/12/5 (0.90/1.30/1.80) but looks like a real
price for id 11 (180.00) — inconsistent semantics in the same column.

The four IDs hardcoded in `constants.js:75-78` all verify: `STANDARD_DELIVERY_ID = 1`,
`NEXT_DAY_DELIVERY_ID = 5`, `SAME_DAY_DELIVERY_ID = 11`, `ECONOMY_DELIVERY_ID = 17`.
**`EXPRESS` (id 12) is not in constants** despite being an enabled, priced option.

### 3.2 Collection depots (live — 6 rows, all enabled)

| id | name | `type` | postcode | whitelist |
|---|---|---|---|---|
| 7 | Bristol Depot | `BRISTOL` | BS3 5RB | |
| 8 | Glasgow Depot | `GLASGOW` | G73 1AF | |
| 9 | Canvey Depot | `CANVEY` | SS8 0QY | |
| 14 | Tipton Depot | `TIPTON` | DY4 7BS | |
| 18 | Nottingham Depot | `NOTTINGHAM` | NG17 1TD | |
| 25 | Head Office, Swadlincote | `HEAD_OFFICE` | DE11 8EA | ✅ |

Matches the depot IDs in `constants.js:80-84` (Glasgow 8, Canvey 9, Bristol 7, Tipton 14).

---

## 4. Time frame — `delivery_time_frame`

The code maps **six** frames to SAP service SKUs (`app/Utils/salesforce-quotes.js:172-203`):

| Time frame | Lead time | `ALL_DAY` | `AM` | `PM` | `TIME_SLOT` |
|---|---|---|---|---|---|
| `STANDARD` | 5 working days | `SERV-DEL-0026` | `SERV-DEL-0060` | `SERV-DEL-0061` | — |
| `EXPRESS` | 3 working days | `SERV-DEL-0014` | `SERV-DEL-0070` | `SERV-DEL-0072` | — |
| `ECONOMY` | 7 working days | `SERV-DEL-0013` | — | — | — |
| `NEXT_DAY` | next day | `SERV-DEL-0005` | `SERV-DEL-0080` | `SERV-DEL-0081` | — |
| `SPECIFIC` | chosen date | `SERV-DEL-0017` | `SERV-DEL-0015` | `SERV-DEL-0016` | `SERV-DEL-00171` |
| `SAME_DAY` | same day | `SERV-DEL-0008` | — | — | — |
| *(collection)* | — | `SERV-COL-0001`, price forced to 0 | | | |

**The database holds 36 distinct values.** `resolveSku` (`salesforce-quotes.js:211-221`)
upper-cases the input, so simple case variants are absorbed; anything else returns `null` and
**no delivery line is added to the Salesforce quote at all.**

Maps successfully (47,791 rows): `STANDARD` 27,938 · `Standard` 3,954 · `EXPRESS` 5,851 ·
`Express` 898 · `express` 5 · `NEXT_DAY` 3,913 · `SPECIFIC` 2,373 · `ECONOMY` 2,370 · `Economy` 256
· `economy` 17 · `SAME_DAY` 216.

**Maps to nothing (~2,777 rows, excluding NULL and collection values):**

| Value | Rows | Why it fails |
|---|---|---|
| `DPD` | 1,628 | a delivery-type code with no SAP SKU |
| `Next Day` | 738 | uppercases to `NEXT DAY` — **space, not underscore** |
| `Specific Day` | 120 | → `SPECIFIC DAY`, not a key |
| `STANDARD_JANUARY_OFFER` | 99 | promo type, no SKU |
| `AUTOMATION_GATE` | 54 | no SKU |
| `STANDARD_FREE` | 49 | no SKU |
| `DIRECT_SUPPLIER` | 20 | no SKU |
| `EXPRESS_JANUARY_OFFER` | 14 | no SKU |
| `08:00 - 10:00` | 10 | ⚠️ a **time window** stored as a time frame |
| `COURIER_SHIPPING` | 7 | no SKU |
| `4_WEEK_ECONOMY_DELIVERY` | 5 | no SKU |
| `1-5 Working day with HIAB offload (container offload only)` | 5 | ⚠️ **free prose** |
| `SPECIALIST_GATES` | 4 | no SKU |
| `HEAVY_BARRIER`, `ECONOMY_DELIVERY`, `CANTILEVER`, `nextDay` | 3 each | no SKU / `NEXTDAY` |
| `CONTAINER_HIAB`, `Without Offload`, `With Hiab Offload`, `00:00 - 02:00` | 1 each | ⚠️ prose / time windows |

Plus `HIRE` 278 and `HIRE DELIVERY AND COLLECTION` 8 (no SKU by design), `NULL` 9,654, and
`Collection`/`collection` 1,227 (collections mis-stored in the frame column).

Values not in the `DELIVERY_TYPES` constant fall through `getTimeFrame`'s `return timeFrame`
default and are rendered **raw** in customer emails — e.g. "Delivery - DPD".

---

## 5. Time slot — `time_type`

Code defines four (`constants.js:56-61`). **The database holds eight:**

| Value | Rows | In the enum? |
|---|---|---|
| `ALL_DAY` | 36,850 | ✅ |
| `NULL` | 11,081 | — |
| `AM` | 10,393 | ✅ |
| `PM` | 1,563 | ✅ |
| `ANY_TIME` | 1,507 | ❌ legacy |
| `TIME_SLOT` | 313 | ✅ |
| `SLOT` | 11 | ❌ |
| `CUSTOM_TIME` | 9 | ❌ (appears only as a code comment) |

`TIME_SLOT` means a 2-hour custom window, priced by `time_modifier` ×1.70 (08:00–18:00) or
`time_off_hours_modifier` ×3.20 outside those hours.

⚠️ **`isCustomTime` is defined as "not `ALL_DAY`, not `AM`, not `PM`"**
(`shipping-services.js:107-110`), so all **1,527** `ANY_TIME` / `SLOT` / `CUSTOM_TIME` rows are
treated as a custom window and rendered as a **2-hour time range** derived from `selected_date`.
For `ANY_TIME` that is semantically backwards — it means *no* window.

Also `getTimeType` has the `AM`/`PM` → "Morning"/"Afternoon" branches **commented out**
(`shipping-services.js:78-83`), so AM/PM render as raw tokens while `ALL_DAY` correctly renders as
"Any Time".

---

## 6. Offload — `offload_type`

**Live rows in `delivery_offloads` (4, all enabled):**

| id | `type` | Name | **Live price** | Seeded price |
|---|---|---|---|---|
| 5 | `SELF_OFFLOAD` | Customer Offload | — (free) | free |
| 6 | `FIRST_FENCE_OFFLOAD` | First Fence Offload | **£108.00** | £89.00 |
| 7 | `ASSISTED_OFFLOAD` | Assisted Offload | **£43.00** | £30.00 |
| 10 | `CUSTOMER_OFFLOAD_MECH` | Customer Offload (Mechanical) | — (free) | free |

Two notes:

- **Live prices differ from `OFFLOAD_PRICES` in `constants.js:69-73`** (which still says 0/30/89).
  **This is harmless — `OFFLOAD_PRICES` is dead code**, declared and never referenced anywhere in
  `app/` or `start/`. (An earlier version of this doc listed the duplication as a divergence risk;
  it is a stale constant, not a pricing bug.)
- **The row IDs are 5/6/7/10, not the seeded 1/2/3/4** — the table was recreated at some point.
  Anything hardcoding an offload ID would be wrong.

**The database holds eight `offload_type` values** — the four above plus three lowercase duplicates:

| Value | Rows |
|---|---|
| `SELF_OFFLOAD` | 39,215 |
| `NULL` | 15,329 |
| `FIRST_FENCE_OFFLOAD` | 2,389 |
| `CUSTOMER_OFFLOAD_MECH` | 1,700 |
| `ASSISTED_OFFLOAD` | 1,555 |
| `self_offload` | 1,391 ⚠️ |
| `assisted_offload` | 105 ⚠️ |
| `first_fence_offload` | 43 ⚠️ |

`delivery_offload_options` layers per-option overrides; `delivery_product_offloads` maps
availability per product.

---

## 7. Supporting tables

| Table | Purpose | Live scale |
|---|---|---|
| `delivery_prices` | **per depot × per type**: `price_per_mile`, `min_price` | 7 depots × 18 types |
| `delivery_variables` | 26 global pricing parameters | §7.1 |
| `delivery_post_codes` | per-type postcode rules | **3,537 rules, 406 blocked prefixes** |
| `delivery_blacklist` | products/types that cannot be delivered | |
| `delivery_whitelist` | used when `enforce_whitelist = 1` — **every** basket product must be whitelisted or the type is dropped | 7 of 16 delivery types enforce it |
| `delivery_blocked_dates` + `delivery_date_type_pivot` | blackout dates per type | |
| `delivery_day_options` | per-type day/slot choices with a `priceModifier` | |

### 7.1 The 26 pricing variables — live values

**Six differ from the repo's seed file.** Live values are authoritative:

| Variable | **Live** | Seeded | |
|---|---|---|---|
| `pounds_per_km` | **1.54** | 1.40 | ⬆ |
| `min_base_price` | **44.00** | 40.00 | ⬆ |
| `min_base_price_weekend` | **165.00** | 150.00 | ⬆ |
| `modifier_saturday` | **3.50** | 3.00 | ⬆ |
| `modifier_sunday` | **5.00** | 4.50 | ⬆ |
| `max_delivery_price` | **2090.00** | 1900.00 | ⬆ |

Unchanged: `two_days_multiplier` 1.30 · `three_days_multiplier` 1.25 · `modifier_bank_holiday` 4.50
· `delivery_lookahead` 80 · `collection_lookahed` 80 *(sic — typo in the stored `type`)* ·
`time_modifier` 1.70 · `time_off_hours_modifier` 3.20 · `min_days_available` 20 ·
`specific_day_modifier` 1.30 · `all_delivery_surcharge` **1.00** · `next_day_increase` **1.00**.

The last two are **1.00 in live data, i.e. currently no-ops** — worth knowing before testing
next-day surcharge behaviour. Seeds mark 9 further variables "Not in use"
(`distance_limit`, `same_day_multiplier`, `next_day_multiplier`, `divide_discount_limit`,
`order_weight_limit`, `depots_order_capacity`, `num_delivery_limit`, `max_extra_charge`,
`london_modifier`) — all still present with values.

### 7.2 Live delivery rates (`price_per_mile` / `min_price` ranges across the 7 depots)

| Type | £/mile | Minimum |
|---|---|---|
| Same Day UK Delivery | 3.50 – 3.80 | 200.00 – 250.00 |
| Next Day Delivery | 1.80 – 2.10 | 160.00 – 200.00 |
| Express Delivery | 1.30 – 1.55 | 50.00 – 90.00 |
| Standard Delivery | 0.90 – 1.20 | 45.39 – 75.00 |
| Heavy Goods Delivery | 0.90 | 45.39 |
| Economy Delivery | 0.88 | 80.00 |
| Turnstile Delivery | 0 | 237.50 |
| DPD Shipping | 0 | 19.50 |
| External Courier | 0 | 14.99 |
| Courier Shipping | 0 | 7.00 |
| **Specific Day Delivery** | **0** | **0** |
| Free Standard Delivery | 0 | 0 |
| all 6 collection depots | 0 | 0 |

Note **Specific Day Delivery has a zero rate and zero minimum** — its price comes entirely from
`specific_day_modifier` (1.30) and day options, not from a base rate. Since it is also the *only*
type that survives the installation filter (§1), that combination is worth a targeted test.

### 7.3 Product-specific delivery rules

`app/Services/delivery/` holds bespoke rule sets, each its own test case:
`cantilever-products.js` · `concreate-products.js` *(sic)* · `envirorail-products.js` ·
`residential-gate-delivery.js` · `steel-road-plate-products.js` · `turnstile-products.js` ·
`hire.js` · `installation.js` · `product-delivery-surcharge.js` · `postcode-delivery-rules.js` ·
`blacklist-by-product.js` · `blockDates.js` · `vehicles.js`.

---

## 8. ⚠️ Case-sensitivity bugs — quantified against live data

Every comparison in the code is a strict `===`. Because the columns are unvalidated `varchar`, the
title-case and lowercase rows in §2/§4/§6 silently take the wrong branch.

### 8.1 Salesforce CPQ quotes lose their delivery line entirely — 7,273 rows (11.8%)

`app/Controllers/Http/SalesforceQuoteWizardController.js:419-422` (and identically at `:712-713`):

```js
if (orderJson?.delivery?.delivery_type === "DELIVERY" ||
    orderJson?.delivery?.delivery_type === "COLLECTION") {
  const deliveryServices = await getDeliveryServices(orderJson, service, quoteId);
  ...
}
```

`Delivery` (6,041) and `Collection` (1,232) match **neither** branch, so `getDeliveryServices` is
never called and the quote carries **no delivery or collection line at all**. Add the 24 `NULL`
and 13 name-leak rows and it is 7,310.

### 8.2 Collections rendered as deliveries in customer emails — 1,237 rows

`shipping-services.js:122` and `salesforce-quotes.js:29,347,353` all test
`=== SERVICES.COLLECTION` (`'COLLECTION'`). The 1,232 `Collection` + 5 `collection` rows fail,
so `getService()` falls through to its "anything else is Delivery" default and the order email
says **"Delivery"** for what is actually a depot collection.

### 8.3 Chargeable offloads dropped from quotes — 148 rows

`getOffloadServices` (`salesforce-quotes.js:281,294,306-308`) compares uppercase literals:

```js
if (delivery.offload_type === "FIRST_FENCE_OFFLOAD")  → SKU SERV-LOA-0001, Target_Price__c = offloadPrice
if (delivery.offload_type === "ASSISTED_OFFLOAD")     → SKU SERV-LOA-0010
if (... === "SELF_OFFLOAD" || ... === "CUSTOMER_OFFLOAD_MECH") → SKU SERV-LOA-0005
```

So `first_fence_offload` (43 rows, **£108 each**) and `assisted_offload` (105 rows, **£43 each**)
add **no offload line to the quote** — roughly £9,159 of chargeable service silently missing across
those 148 orders. `self_offload` (1,391) also misses, but that SKU is free so the impact is
cosmetic.

### 8.4 Other confirmed mismatches

- **`Next Day` → no SAP SKU (738 rows).** `resolveSku` uppercases to `NEXT DAY`; the map key is
  `NEXT_DAY`. A legitimate next-day delivery silently produces no delivery SKU. Same for
  `Specific Day` (120) and `nextDay` (3).
- **1,527 rows render a fake 2-hour delivery window** (§5) because `ANY_TIME`/`SLOT`/`CUSTOM_TIME`
  fall into the `isCustomTime` branch.
- **Two columns for one concept.** `delivery_type` (string service) and `delivery_type_fk` (FK) are
  both written on every order. Admin filtering uses the FK join
  (`OrderController.js:276-284`); emails and the Salesforce mapping use the string.
- **`DELIVERY_TYPES` constant lists 3 of the 36 stored values**; `OFFLOAD_TYPES` lists 3 of the 4
  real offload types (`CUSTOMER_OFFLOAD_MECH` is absent, though it has 1,700 rows).
- **No validation anywhere.** All four columns are `varchar(256)` with no enum constraint and no
  validator (`OrderController.js:522-533` writes whatever the client sends) — which is how prose
  like `"1-5 Working day with HIAB offload (container offload only)"` and time strings like
  `"08:00 - 10:00"` ended up in a time-frame column.

**Suggested fix direction:** normalise on read (`String(x).toUpperCase().replace(/ /g,'_')`) in
`shipping-services.js` and `salesforce-quotes.js`, add a validator on write, then backfill. The
`resolveSku` function already uppercases — it just doesn't handle separators.

---

## 9. Endpoints

**Public** (`start/routes/public.js`)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/delivery` | main quote — returns `deliveryTypes`, `collectionTypes`, `deliveryDates`, offloads. `GetDelivery` validator |
| `POST` | `/validate/postcode` | → `{ code: "VALID_POSTCODE", postcode }` or `404` |
| `POST` | `/reverse-lookup/postcode` | coordinates → postcode. ⚠️ **500s on every successful call** — see the 5xx doc |
| `POST` | `/delivery/blocked-collection-products` | products that cannot be collected |
| `GET` | `/delivery/blacklist/by-product/:id` | blacklist for one product |

**Admin** (`start/routes/admin.js:108-178`) — full CRUD on `/delivery/types`, `/delivery/postcodes`,
`/delivery/prices`, `/delivery/blacklist`, `/delivery/whitelist`.

---

## 10. Code reference index

- `app/Utils/constants.js:36-84` — `SERVICES`, `DELIVERY_TYPES`, `TIME_SLOT`, `OFFLOAD_TYPES`,
  `OFFLOAD_PRICES` (dead), the four delivery-type IDs, depot IDs
- `app/Utils/shipping-services.js` — renders the combination into the customer-facing string;
  `:10-15` the DELIVERY fallback, `:78-83` commented-out AM/PM, `:107-110` `isCustomTime`, `:122`
  the `COLLECTION` comparison
- `app/Utils/salesforce-quotes.js:172-203` (SKU map), `:211-221` (`resolveSku`), `:224-265`
  (`getDeliveryServices`), `:278-315` (`getOffloadServices`)
- `app/Controllers/Http/SalesforceQuoteWizardController.js:419-422, 712-713` — the quote gate
- `app/Services/delivery/deliveryTypes.js` — delivery/collection split, whitelist/blacklist
  filtering, `nextDayDeliveryPriceIncrease`
- `app/Services/delivery/installation.js:8-18` — the `SPECIFIC`-only filter
- `app/Services/delivery/hire.js:22-73` — hire day types and doubling
- `app/Services/delivery/vehicles.js` — rigid/artic capacities
- `app/Controllers/Http/DeliveryController.js` — the public quote endpoint
- `app/Controllers/Http/OrderController.js:522-533` — where the five axes are persisted
- Migrations: `1571815706410_delivery_types_schema.js`, `1571815706411_delivery_offload_schema.js`,
  `1571815706412_delivery_schema.js`, `1571815680716_delivery_variables_schema.js`,
  `1617804233970_delivery_post_codes_schema.js`, `1617804301560_delivery_prices_schema.js`,
  `1617965958493_delivery_blacklist_schema.js`, `1619181609051_delivery_offload_options_schema.js`,
  `1619514130943_delivery_day_options_schema.js`, `1665666419641_delivery_blocked_dates_schema.js`,
  `1665667769554_delivery_date_type_pivot_schema.js`, `1667481534964_delivery_whitelist_schema.js`,
  `1745502902752_delivery_product_offload_schema.js`
- Seeds (**now stale** vs live — see §6, §7.1): `database/seeds/DeliveryOffloadSeeder.js`,
  `database/seeds/DeliveryParamSeeder.js`

### Queries used

```sql
SELECT id,name,type,days,collection,enabled,enforce_whitelist,is_google_shopping,price,sort_order
  FROM delivery_types WHERE is_deleted=0 ORDER BY collection,sort_order,id;
SELECT CAST(delivery_type AS BINARY) v, COUNT(*) n FROM deliveries GROUP BY v ORDER BY n DESC;
SELECT CAST(delivery_time_frame AS BINARY) v, COUNT(*) n FROM deliveries GROUP BY v ORDER BY n DESC;
SELECT CAST(time_type AS BINARY) v, COUNT(*) n FROM deliveries GROUP BY v ORDER BY n DESC;
SELECT CAST(offload_type AS BINARY) v, COUNT(*) n FROM deliveries GROUP BY v ORDER BY n DESC;
SELECT id,type,name,price,enabled FROM delivery_offloads;
SELECT type,value FROM delivery_variables ORDER BY id;
SELECT dt.name,COUNT(*),MIN(dp.price_per_mile),MAX(dp.price_per_mile),MIN(dp.min_price),MAX(dp.min_price)
  FROM delivery_prices dp JOIN delivery_types dt ON dt.id=dp.delivery_type_id
  WHERE dp.is_deleted=0 GROUP BY dt.name;
SELECT COUNT(*),SUM(blocked) FROM delivery_post_codes;
```

`CAST(... AS BINARY)` is essential — the columns use a case-insensitive collation, so a plain
`GROUP BY` silently merges `DELIVERY` with `Delivery` and hides every bug in §8.
