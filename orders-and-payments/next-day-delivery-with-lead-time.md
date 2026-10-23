# Next Day delivery is offered for products with a long lead time: why, and is it a bug?

QA asked: *"Why can you choose Next Day delivery on products with a long lead time?"* QA also
said the website works the same way. This doc answers both, puts numbers on how often it
happens, and lists what each team would have to change.

**Data provenance:** dev MySQL (`website_test` @ `54.171.181.199`), read-only, on **2026-10-23**
(server time 08:38 UTC). Catalogue counts come from the **dev GraphQL**
(`https://cdn.dev.firstfence.co.uk/graphql`, one read-only `allProducts` query) on the same day.
The dev Mongo (`cdn-test` @ `ec2-34-252-129-247`) did not accept connections on port 27017 from
this machine, so I could not query it directly (see §12). Code read at:

| Repo | Ref | Date | Local refs fetched |
|---|---|---|---|
| website-api | `mobile-app` `8be170f`; `origin/master` `d27b5cc`; `origin/dev-docker` `a183a69` | 2026-10-22 | 2026-10-22 17:46 |
| gatsby-website | `master` `b8b24861e`; `origin/test-site` `b806887ee` | 2026-10-14 | 2026-10-15 17:28 |
| ff-uk-mobile | `dev` `94e2b87` | 2026-10-21 | 2026-10-22 17:07 |
| admin-website-v2 | `mobile-app` `5f42d3c` | 2026-10-22 | 2026-10-22 17:35 |

The dev website build is from **22 Oct 2026 13:51 GMT** (`last-modified` of
`/page-data/app-data.json`). It is current. No writes of any kind were made. Production was not
touched.

Companion docs: [`delivery-types-and-options.md`](./delivery-types-and-options.md) (all delivery
types and fields), [`../pdp-research/LEAD_TIME_BADGE_EXPLAINED.md`](../pdp-research/LEAD_TIME_BADGE_EXPLAINED.md)
(where the lead-time number comes from).

---

## TL;DR

**Yes, it is a real case, and QA is right: the website does the same as the app.** No code in any
repo checks the basket's lead time before it offers Next Day, Same Day or Express. The check was
never built. It is not a regression. Production has the same behaviour: the production copy on dev
holds **78 completed Next Day orders** and **5 completed Same Day orders** that contained an item
with a lead time (April 2023 to June 2026).

Is it a bug? **That is a business decision, because the system already has one model built in:
"Next Day" means one working day after the goods are ready. The lead time is added on top.** A Next
Day order with a 10-day item is expected in 11 working days, not tomorrow. The team confirmed this
model for the app's expected date on 2026-09-30 (FIR-266). Nobody ever decided the other way: no
doc, ticket or commit says Next Day should be hidden for lead-time baskets.

Some things are wrong whichever model the business chooses:

| # | Problem | Where | Whose repo |
|---|---|---|---|
| 1 | **The website never shows a date** for Next Day, Same Day, Express or Standard. The shopper sees "Next Day Delivery" and a price. Nothing tells them the item must arrive in the warehouse first. | gatsby-website checkout and basket | website (not yours) |
| 2 | The app **does** show the correct date ("Estimated delivery", lead time included). But the option is still called "Next Day Delivery". | ff-uk-mobile checkout | mobile (not yours) |
| 3 | **The Salesforce quote says "tomorrow".** `Due_Date__c` for a Next Day order is the push date + 1 working day. The lead time goes into a separate field, `Lead_Time_In_Days__c`. Nothing combines the two. | `app/Utils/salesforce-quotes.js:85-139` | **website-api (yours)** |
| 4 | The customer pays the Next Day price. The average delivery charge is **£214.16** on Next Day orders and **£109.33** on Standard orders. On the 78 orders this is **about £8,200 more than Standard** (an estimate from averages, see §5.3). | data | business |

Key numbers:

| What | Number |
|---|---|
| Buyable products with a lead time (dev catalogue, today) | **1,642 of 3,540 (46.4%)**. The shortest lead time is 2 working days, so for every one of them Next Day cannot mean "tomorrow". |
| Completed real orders, Next Day + a lead-time item (Apr 2023 to Jun 2026) | **78**, £16,852.65 delivery charged, lead times 3 to 25 working days |
| Completed real orders, Same Day + a lead-time item | **5**, £1,364.41, lead times 5 to 7 |
| Completed real orders, Express (1-3 days) + an item with lead time > 3 days | **256**, £39,450.59, lead times 5 to 35 |
| Dev test orders with this pattern since 4 Sep 2026 | **16** (7 Next Day, 6 Same Day, 3 Express), including QA's #60862 |

**Recommendation:** fix #3 in website-api now. It is wrong under both models. Then ask the
business one question (§4). If they want Next Day and Same Day hidden, option A2 in §9 is the
reversible way to do it.

---

## 1. The literal question: why can you choose it?

Because nothing removes it. Both the website and the app call `POST /delivery`. The server builds
the option list in `DeliveryController.initialize`. It does work out the basket's lead time, but it
uses the number for dates only, never for the list of delivery types.

1. `findLeadTimeDays(products)` takes the longest `leadTimeDuration` of all basket lines
   (`app/Services/delivery/products.js:165-175`; called at `DeliveryController.js:472`).
2. The delivery type list comes from `getDeliveryAndCollectionTypes`
   (`app/Services/delivery/deliveryTypes.js:22-165`). Its inputs are depot prices, blacklist,
   whitelist, postcode rules and `enabled`. **Lead time is not an input.**
3. After that, the list is filtered by installation (`DeliveryController.js:545-548`), weight
   (`:553-578`), blocked dates (`:582-593`), and time-of-day cut-offs (`:860`,
   `delivery-cutoffs.js:85-121`). **None of them reads lead time.**

What the lead time does change today:

| Thing | Changed by lead time? | Code |
|---|---|---|
| List of delivery types (Standard, Express, Next Day, Same Day, Specific Day) | **No** | `deliveryTypes.js:22-165`, `DeliveryController.js:481-593` |
| Price of a delivery type | **No** | same |
| Specific Day calendar: dates inside the lead time are flagged `leadTime: true` | Yes | `dates.js:323-325` |
| Specific Day calendar: the calendar gets longer so enough dates stay free | Yes | `dates.js:247-255` |
| Collection dates: flagged the same way | Yes | `dates.js:38-45`, `dates.js:76-78` |
| Glasgow depot: blocked when a product has a "Finish" option **and** a lead time | Yes | `blacklist.js:8-20` (since `695c08e`, 2021-05-13) |
| Date stored on the order for Next Day | No date is stored at all | §3 |

So **inside one checkout, Specific Day respects the lead time** (the shopper cannot pick a date
inside it), **but Next Day, Same Day and Express do not.** QA is seeing this difference.

The list of the five public delivery types on dev today (`delivery_types`):

| id | name | `type` | `days` | `sort_order` | `details` shown to the shopper |
|---|---|---|---|---|---|
| 17 | Economy Delivery | `ECONOMY` | 7 | 0 | "Delivery anytime between 1-7 working days." (only shown when cheaper than Standard) |
| 1 | Standard Delivery | `STANDARD` | 5 | 1 | "Delivery anytime between 1-5 working days." |
| 12 | Express Delivery | `EXPRESS` | 3 | 2 | "Delivery anytime between 1-3 working days." |
| 5 | Next Day Delivery | `NEXT_DAY` | 1 | 3 | "Available on orders placed before 2.30pm, delivery on working days only. Orders placed over the weekend will not be processed until the following working day." |
| 11 | Same Day UK Delivery | `SAME_DAY` | 0 | 4 | "Available only on orders placed before 10am, **subject to stock availability**." |
| 10 | Specific Day Delivery | `SPECIFIC` | 0 | 5 | "Select day and time of delivery" |

Only the Same Day text mentions stock. The Next Day text does not.

---

## 2. "On the website it works the same": confirmed, with one difference

| | Website (gatsby-website) | App (ff-uk-mobile) |
|---|---|---|
| Call | `POST /delivery` with `{ postcode }` only (`src/pages/checkout/delivery/index.js:126-129`) | `POST /delivery` with `{ postcode }` only (`src/features/delivery/services/deliveryApi.ts:34-35`) |
| Where the server gets the basket | The server cart in MySQL (`CartProduct` middleware, `app/Middleware/CartProduct.js:23-29,60-61`) | same |
| Shows every type the server returns | Yes. Only Google Shopping types are removed (`index.js:141-143`). `delivery.map(...)` at `src/features/checkout/service-page/delivery-options/delivery-options.js:29-42` | Yes. Only Google Shopping types are removed (`src/features/delivery/utils/deliveryQuote.ts:118,159`) |
| Hides, disables or warns about Next Day for lead time | **No.** Nothing in `src`, also no commented-out code | **No.** Nothing in `src`, also no commented-out code |
| **Shows a date for Next Day before payment** | **No.** The option card shows `name`, price, `subtitle`, `details` only (`delivery-option/delivery-option.js:17-38`). The basket panel shows name and price only (`estimate-delivery/delivery-info/delivery-info.js:42`). | **Yes.** "Estimated delivery" = today + 1 working day + the basket's longest lead time (`src/features/delivery/utils/deliveryEstimate.ts:79-96`). A "Why this date?" sheet lists the lead-time items (`src/ui/components/WhyDateSheet.tsx:51-55`). |
| Lead time in the Specific Day calendar | Dates with `leadTime` are disabled and labelled "Lead Time" (`src/features/checkout/service-page/calendar/date-tiles.js:58`) | Dates with `leadTime` are removed (`src/features/delivery/utils/availableDates.ts:90`) |
| Order payload for Next Day | `timeFrame: "NEXT_DAY"`, no `date` (`src/pages/checkout/contact-details/index.js:214-215`; `date` is set only when a calendar date was picked, `:233-235`) | `timeFrame: "NEXT_DAY"`, `date: null` (`src/features/checkout/utils/draftDeliveryDate.ts:33`) |
| Lead time stored on the basket line | Worked out when the item is added: max(product, selected option) + working days until `dateDueBackInStock` (`src/utils/product/lead-time.js:34-73`, sent from `src/models/Product.js:688,759-760`) | Same rule (`src/core/pricing/configurator.ts:244-248`). Exception: extra/upsell lines use the product's base value only (`src/features/product/utils/basketLines.ts:150`). |

Two more facts:

- **The shopper chooses Next Day on purpose.** The website pre-selects the first option
  (`index.js:147-148`). With the `sort_order` values above, that is Standard, or Economy when
  Economy is cheaper. Next Day is never pre-selected unless all the types before it are missing.
- **Production website = dev website here.** gatsby-website `master` deploys to production and
  `test-site` deploys to the dev site (`.gitlab-ci.yml:8-9`, `:111-112`). Their checkout code
  differs only in a BT-postcode message and the customer note. There is no lead-time difference.

---

## 3. What happens after the order is placed

The example is QA's dev order **#60862**: Next Day, placed Sun 27 Sep 2026 10:58 UTC, two EnduraMat
couplers with a 10-day lead time.

| Place | What it says | Code |
|---|---|---|
| `deliveries` row | `delivery_time_frame = NEXT_DAY`, `delivery_type_fk = 5`, `selected_date = NULL`. There is no lead-time column. The lead time lives only on the order's lines (`products.leadTimeDuration`, linked by `products.delivery_ID`). | `OrderController.js:570-581, 594-598` |
| App order list (`fulfilment.expectedDate`) | **13 Oct 2026**: created date + 1 + 10 working days. On `dev-docker` and `mobile-app` only, **not on `master`**. | `app/Utils/expected-delivery-date.js:271-282, 404-431` (`2219d03`, 2026-09-30, FIR-266) |
| Website customer portal | Adds the longest line lead time to the service days. This is display only. | gatsby `src/utils/portal/order-delivery.js:99-133` |
| Salesforce quote (pushed from the admin) | `Due_Date__c` = **push date + 1 working day**. `Lead_Time_In_Days__c` = 10, as a separate field. | `salesforce-quotes.js:85-139` (`getDueDate`, starts from `Date.now()` at `:89`, `NEXT_DAY: 1` at `:125`); `SalesforceQuoteWizardController.js:304,348` and `:605,649` |
| SAP | Not checked: SAP is production only. When an order is linked, the app uses SAP's `ORDR.DocDueDate` (FIR-260), so staff can correct the date in SAP. | `expected-delivery-date.js:341-431` |

So **after checkout, the system already treats the lead time as extra days on top of "Next Day".**
The website portal and the app both do this. Only the Salesforce due date does not.

---

## 4. Is it a bug? Two possible meanings of "Next Day"

| | A. "Transport speed" (what the code assumes) | B. "Arrives tomorrow" (what the name says to a shopper) |
|---|---|---|
| Meaning of "Next Day Delivery" | One working day **after the goods are ready** | Tomorrow |
| What the system should do | Keep offering it. Show the real date **before** payment. | Do not offer Next Day or Same Day when any basket line has a lead time. |
| What is wrong today | Website shows no date (#1). Salesforce due date (#3). The name may mislead. | It is offered in every channel. |

Evidence that the team works with meaning A today:

- `website-api/.generated_docs/FirstFence-Meetings.md:340-345` (Q13): *"Everything ships together
  — no partial dispatch … Estimated delivery date = order date + standard delivery days + longest
  lead time of any item in the basket."*
- FIR-266 decisions, 2026-09-30
  (`website-api/.generated_docs/payments-bugs-2026-09-28/FIR-266-verification.md`): the app's
  expected date counts the lead time on top of the service days for every order. The ticket was
  about the **date shown**, not about which services are offered.
- The app checkout shows today + 1 + lead time for Next Day (§2).

Under meaning A, does the extra money buy anything? Next Day + a 10-day item = 11 working days.
Standard + the same item = up to 15 working days. So the customer gets the goods up to 4 working
days sooner, **but only if the warehouse sends the order out on the day the stock arrives.** I could
not check that (it needs SAP, which is production only). For Same Day + a lead time, the transport
team must confirm that "same day after the stock arrives" is a promise they can keep.

**The question for the business:**

> When a basket contains an item with a lead time, should we still offer Next Day and Same Day
> (and Express)? If yes, the shopper sees "Next Day" but gets "lead time + 1 working day". Is that
> what we want to sell, and should the option say so?

---

## 5. The numbers

### 5.1 Catalogue (dev, 2026-10-23)

"Buyable" = `status = PUBLISHED`, `visible` not false, `productType` not `GROUP` or `POA`. This is
the same visibility rule the website build uses (`gatsby-node.js:89`, `visible: { ne: false }`).

| | Products |
|---|---|
| Buyable products | 3,540 |
| with `leadTimeDuration > 0` | **1,642 (46.4%)** |
| shortest lead time among them | 2 working days, so **all 1,642** make "Next Day = tomorrow" impossible |
| lead time ≥ 5 days | 1,567 |
| lead time ≥ 10 days | 1,215 |
| lead time ≥ 20 days | 679 |

This does not count option-level lead times. A product that is in stock can get a lead time when
the shopper picks an option (for example a RAL colour). The badge doc measured 727 such products on
production; I did not measure it again.

### 5.2 Real orders: the production copy on dev

The dev database holds a copy of production. Orders up to **2026-06-24 ~12:28 UTC** are real
customer orders (60,299 is about the last id). Several tables were created at 2026-06-24 12:28.
Order line rows (`products`) start on **2023-04-03**, so the window is 2023-04-03 to 2026-06-24.

"Affected" = the longest line `leadTimeDuration` is longer than the service's own `days`
(Same Day 0, Next Day 1, Express 3).

| Type | Orders with lines | Affected (any status) | Affected and **completed** | Delivery charged on completed | Lead time range | Completed with lead ≥ 10 |
|---|---|---|---|---|---|---|
| Next Day | 2,029 | 84 | **78** | **£16,852.65** | 3 to 25 | 10 |
| Same Day | 111 | 5 | **5** | **£1,364.41** | 5 to 7 | 0 |
| Express | 3,028 | 270 | **256** | **£39,450.59** | 5 to 35 | 65 |

The other 6 Next Day orders: 2 cancelled, 2 pending, 1 closed, 1 test. For reference, 1,347
Standard orders had a lead time longer than 5 days. Standard is the default choice, so those are
not part of this question.

By year (any status):

| Year | Next Day orders | of which affected | Same Day orders | of which affected |
|---|---|---|---|---|
| 2023 (from 3 Apr) | 452 | 34 | 29 | 3 |
| 2024 | 597 | 25 | 37 | 1 |
| 2025 | 675 | 18 | 33 | 0 |
| 2026 (to 24 Jun) | 305 | 7 | 12 | 1 |

It happens every year. It is about 4% of Next Day orders.

### 5.3 The price difference (estimate)

Average delivery charge per order, same window, all orders with lines:

| Type | Average `deliveries.price` | Orders |
|---|---|---|
| Standard | £109.33 | 13,811 |
| Express | £145.15 | 3,028 |
| Next Day | £214.16 | 2,029 |
| Same Day | £264.54 | 111 |

Estimate of the extra paid compared with Standard, on the completed affected orders:

| Type | Extra per order | Orders | Extra in total |
|---|---|---|---|
| Next Day | £104.83 | 78 | **≈ £8,177** |
| Same Day | £155.21 | 5 | **≈ £776** |

This is an estimate from averages. The real Standard price depends on each order's distance and
depot, and I did not recompute it per order.

### 5.4 Dev test orders (after the production copy)

16 orders since 4 Sep 2026 have a fast service and a lead-time line:

| Order | Created (UTC) | Type | Delivery price | Longest lead time |
|---|---|---|---|---|
| 60488 | 2026-09-04 13:10 | SAME_DAY | £266.80 | 30 |
| 60555 | 2026-09-09 20:57 | SAME_DAY | £500.00 | 30 |
| 60575 | 2026-09-10 15:44 | SAME_DAY | £500.00 | 5 |
| 60736 | 2026-09-23 08:46 | SAME_DAY | £3,077.57 | 10 |
| 60860 | 2026-09-27 09:18 | SAME_DAY | £500.00 | 10 |
| 60861 | 2026-09-27 09:18 | EXPRESS | £153.50 | 10 |
| **60862** | 2026-09-27 10:58 | **NEXT_DAY** | £214.00 | 10 |
| 60863 | 2026-09-27 11:20 | EXPRESS | £153.50 | 10 |
| 60993 | 2026-10-18 17:33 | NEXT_DAY | £214.00 | 10 |
| 60994 | 2026-10-18 17:36 | EXPRESS | £153.50 | 10 |
| 60998 | 2026-10-20 20:37 | NEXT_DAY | £500.00 | 10 |
| 60999 | 2026-10-20 20:58 | NEXT_DAY | £500.00 | 10 |
| 61001 | 2026-10-20 21:06 | NEXT_DAY | £500.00 | 10 |
| 61018 | 2026-10-22 08:44 | SAME_DAY | £500.00 | 10 |
| 61020 | 2026-10-22 10:36 | NEXT_DAY | £160.00 | 10 |
| 61026 | 2026-10-22 15:58 | NEXT_DAY | £225.00 | 10 |

The Next Day orders at £500 were placed after 15:00 UK time. After 3pm the backend gives Next Day
the Same Day price (`delivery-cutoffs.js:97-113`). That is expected and not related to lead time.

---

## 6. Dating it: the check never existed

| Date | Commit | What changed |
|---|---|---|
| 2019-10-02 | web `4103f0b20` "delivery system finished" | The old checkout (`bookDelivery.js`) had `isLeadTime`. When a basket had both in-stock and lead-time dates, it offered a **split delivery** (DELIVERY A / DELIVERY B). It did not hide delivery types. |
| 2020-12-03 | web `8da0cbd0d` "start checkout" | The current checkout was started. |
| 2021-01-07 | web `00d174364` "refactoring" | The old checkout file was deleted. The split-delivery prompt went with it. |
| 2021-04-29 | api `93dc188` "delivery system update", web `ced93efcf` | Today's `date.leadTime` calendar flags and the disabled calendar tiles. |
| 2021-05-13 | api `695c08e` | Glasgow collection blocked for lead-time products with a Finish option. |
| 2022-05-12 | api `05c47c9` "remove leadtime required" | Cart lines no longer need a lead time to be valid. |
| 2025-07-30 | web `41cfee55e` | Next Day cut-off note at **2:30pm**. It moved to 10pm on 2026-08-07 (web `8ca1a50c8`, api `150ef7e`). It is about time of day, not lead time. |
| 2026-09-17 | api `18ae389` (sc-1539) | 10am Same Day and 3pm Next Day cut-offs. About time of day, not lead time. |
| 2026-09-26 | app `10077b3` | App dates count working days **and** lead time. |
| 2026-09-30 | api `2219d03` (FIR-266) | App expected date adds lead time. Not on `master`. |

`git log -G leadTime` on `DeliveryController.js` and `app/Services/delivery/` finds 13 commits,
from `93dc188` (2021-04-29) to `18ae389` (2026-09-17). Every one only changes calendar
bookkeeping (lookahead, `date.leadTime`, `leadTimeDays++`) or the Glasgow rule. **None filters
delivery types, and none removes such a filter.** The same is true for gatsby-website and
ff-uk-mobile. The production data agrees: the pattern is in every year since line data starts.

---

## 7. Other possible causes, ruled out

| Suspect | Why it is not the cause |
|---|---|
| The dev website is an old build | Built 22 Oct 2026. The app does the same, and so do production orders. |
| The lead time does not reach the server | It does. #60862's lines hold `leadTimeDuration = 10`, and the same request flags the Specific Day calendar dates as lead time. |
| A blacklist or whitelist was set up wrong | The blacklist is manual, per product (optionally per option). It never reads lead time (`blacklist-rows.js:21-39`). No row links lead time to a delivery type. |
| A missing setting on the delivery type | The admin delivery type form has no lead-time field: Type, Name, Subtitle, Postcode, Enabled, Collection, Enforce Whitelist, Details, Days, Sort order, Latitude, Longitude, Weight Limit, Max Weight (`admin-website-v2/src/pages/content/delivery/delivery-types/delivery-type-page/` `general-fields/general-fields.jsx:47-160`). `delivery_types` has no such column either. |
| The 10am / 3pm cut-offs | They depend on the time of day only. After 3pm, Next Day loses AM/PM and gets the Same Day price. After 10am, Same Day is removed (`delivery-cutoffs.js:85-121`). |
| The installation filter | With installation, only Specific Day is left (`installation.js:8-18`), and Specific Day respects lead time. So **installation baskets are already safe**. |
| A dev-only change | `origin/master` and `origin/dev-docker` handle lead time the same way in `DeliveryController.js` and `app/Services/delivery/`. Only doc comments differ. |

---

## 8. "Is this just dev?": three separate answers

| Question | Answer |
|---|---|
| Can a deploy or pipeline cause this on prod? | It is not caused by a deploy. It is how the code has worked since 2021, on `master` and `dev-docker` alike (website-api), and on `master` and `test-site` alike (gatsby-website). |
| Can it happen again after the fixes already in the repo? | Yes. There is no fix on any branch. FIR-266 (`2219d03`) changed only the date the app shows after the order. |
| Is prod already in that state? | **Yes.** The production copy holds 78 completed Next Day and 5 completed Same Day orders with lead-time items (§5.2). Orders after 2026-06-24 are not in the copy. To count them, run the query below on production yourself. |

Read-only query for production (MySQL, `website` database). It counts completed orders since the
copy was taken:

```sql
SELECT dt.type, COUNT(*) AS orders, ROUND(SUM(x.dprice), 2) AS delivery_charged,
       MIN(x.lt) AS min_lead, MAX(x.lt) AS max_lead
FROM (
  SELECT o.id, d.delivery_type_fk AS fk, d.price AS dprice,
         MAX(COALESCE(p.leadTimeDuration, 0)) AS lt
  FROM orders o
  JOIN deliveries d ON d.id = o.delivery_A
  JOIN products p ON p.delivery_ID = d.id
  WHERE o.created_at >= '2026-06-24' AND o.status = 1
  GROUP BY o.id, d.delivery_type_fk, d.price
) x
JOIN delivery_types dt ON dt.id = x.fk
WHERE dt.type IN ('NEXT_DAY', 'SAME_DAY', 'EXPRESS') AND x.lt > dt.days
GROUP BY dt.type;
```

---

## 9. If the business wants to change it: options and traps

| Option | What | Repo | Reversible? | Fits meaning |
|---|---|---|---|---|
| **C. Salesforce due date** | Add the order's longest lead time in `getDueDate` | website-api (yours) | Yes, code only | **A and B** |
| **B. Show the date on the website** | Show "Estimated delivery" per option, like the app | gatsby-website | Yes, code only | A |
| **A1. Hide in the backend, fixed rule** | In `/delivery`, drop `SAME_DAY` and `NEXT_DAY` when `leadTimeDays > 0` | website-api (yours) | Yes, code only | B |
| **A2. Hide in the backend, admin setting** | New nullable column `delivery_types.max_lead_time_days`. `NULL` = no limit (today's behaviour). Drop a type when the basket lead time is above its limit. | website-api + admin-website-v2 | Yes. Nothing changes until someone sets a value. Rollback drops the column. | B, or anything between A and B |
| D. Admin delivery blacklist per product | Blacklist Next Day on each lead-time product | admin only | Yes | B |

**Option C, `getDueDate`** (`salesforce-quotes.js:85-139`): `findLeadTimeDays` already exists in
the same file (`:329-339`). Trap: `getDueDate` counts from `Date.now()` (the moment the admin pushes
the quote), not from the order's `created_at`. That is a separate, older issue. Decide whether to fix
it in the same change.

**Options A1 and A2, where to put the filter:** right after `getDeliveryAndCollectionTypes`
(`DeliveryController.js:481-494`). Then the concrete, steel road plate, cantilever and turnstile
branches get the filtered list too. Container, Hire, Heavy Barrier and residential gate build their
own lists and do not offer Next Day, so they are not affected. Traps:

1. **Cut-off rule.** After 15:00, `applyDeliveryCutoffs` copies the Same Day price into Next Day.
   If there is no Same Day, it **removes Next Day** (`delivery-cutoffs.js:97-113`). So if you hide
   Same Day but keep Next Day for lead-time baskets, Next Day also disappears every afternoon.
   Decide the two types together.
2. **The lead time is old and client-supplied.** The number on each cart line is whatever the client
   sent when the item was added (`CartController.js:397-398, 432-433`). It is not read again from
   Mongo (`CartProduct.js:23-29,60-61`). The `dateDueBackInStock` part goes out of date as days
   pass. A filter would use this old number. The Specific Day calendar already has the same
   limitation.
3. **Two rules for "has a lead time".** `findLeadTimeDays` ignores the `leadTime` flag
   (`products.js:168`). `orderLeadTimeDays` counts a line only when the flag is set
   (`expected-delivery-date.js:271-282`). In the data they agree on every order I counted. Still,
   pick one rule.
4. **The order endpoint trusts the client.** `/order/create` saves whatever delivery the client
   sends (`OrderController.js:570-581`). The `StoreOrder` validator is commented out
   (`start/routes/public.js:38-39`). Hiding Next Day in `/delivery` does not stop a client from
   posting it. If the business wants a hard rule, check it in `OrderController.save` too.
5. **Saved basket quotes.** A quote saved with Next Day converts as it is
   (`BasketQuoteController.js:1966-1977` copies the delivery). A filter in `/delivery` does not
   touch saved quotes.
6. **A2 admin plumbing.** `DeliveryTypeController.store` and `update` read each field by name
   (`:11-45`, `:62-110`), as they do for `weight_limit` and `max_weight`. A new column must be added
   in both places, or the admin will drop it without any error. The admin form needs a new input in
   `general-fields.jsx`.
7. **No frontend change is needed to hide an option.** Both the website and the app show exactly
   what the server returns. The website pre-selects `delivery[0]` (`index.js:147-148`), and Standard
   stays in the list, so the list never becomes empty because of this filter.

**Option D** needs no code, but it is manual for 1,642 products, and it does not follow lead-time
changes. A product that comes back into stock stays blocked. Not recommended.

---

## 10. Reconciliation with existing docs

| Doc | Status |
|---|---|
| [`delivery-types-and-options.md`](./delivery-types-and-options.md) (2026-08-20) | Still correct. One wording trap: its §4 table has a column called "Lead time" that means the **service's** working days (Standard 5, Express 3). This doc uses "lead time" for the **product's** `leadTimeDuration`. They are different things. That column would be clearer as "Service days". |
| [`../pdp-research/LEAD_TIME_BADGE_EXPLAINED.md`](../pdp-research/LEAD_TIME_BADGE_EXPLAINED.md) | Still correct. Its production figure (about 46% of buyable products have a lead time) matches dev today (46.4%). It covers the product page only, not checkout. |
| `website-api/.generated_docs/FirstFence-Meetings.md:350-351` (Q13) | **Partly wrong for the website.** It says that at basket and checkout level the customer sees *"just a single delivery date"*. On the website the customer sees **no** date for Next Day, Same Day, Express, Standard or Economy before payment. Only Specific Day shows dates. The formula at `:343` is still correct. |
| `website-api/.generated_docs/payments-bugs-2026-09-28/FIR-266-verification.md` | Correct when written. Its "What the backend does" section (no lead time in `expectedDate`) describes the state before `2219d03`. On `dev-docker` the backend now adds the lead time. On `master` it still does not, because the `fulfilment` block is not on `master`. |
| `QA-Order-Status-Guide.md`, `App-Status-Values-Reference.md`, `App-Statuses-Short-List.md` (website-api) | Correct for `dev-docker` / `mobile-app`: the expected date adds the longest lead time. |

---

## 11. Related findings (not the cause, but found on the way)

1. **Three different Next Day cut-off times.** The Next Day `details` text in `delivery_types` says
   **"before 2.30pm"**. The website warns after **22:00**
   (`src/features/checkout/sidebar/delivery/delivery.js:39-44`). The app uses **22:00** too. The
   backend switches Next Day to rush pricing at **15:00** (`delivery-cutoffs.js:15`). The 10pm
   rule was agreed as "10pm (excluding Scotland)", and the Scotland part was never built. See
   [`scotland-delivery-cut-offs-and-dates.md`](./scotland-delivery-cut-offs-and-dates.md).
2. **App extras use the base lead time.** Extra/upsell lines in the app use the linked product's
   base `leadTimeDuration` with no option or `dateDueBackInStock` adjustment
   (`ff-uk-mobile/src/features/product/utils/basketLines.ts:150`).
3. **The 2026-06-24 copy line.** Anyone who counts "real" orders on dev must stop at about
   2026-06-24 12:28 UTC (order id about 60,299). After that, the rows are dev test orders.

---

## 12. What I did NOT verify

| Item | Why |
|---|---|
| Whether the warehouse really ships a Next Day lead-time order on the day the stock arrives | Needs SAP (`ORDR.DocDueDate`, delivery notes). SAP is production only. |
| Whether sales staff call these customers or refund the Next Day premium | Not in the data I read. `order_details.called` was not analysed. |
| Production orders after 2026-06-24 | Production was not touched. Query in §8. |
| The catalogue directly in dev Mongo | Port 27017 on `ec2-34-252-129-247` did not answer from this machine. I used the dev GraphQL instead. **I assume** it reads the same `cdn-test` database. Its newest `updated` value is 2026-10-20, so it is a live, edited database, not the June production dump. |
| A live `POST /delivery` on the dev API | The website and app send only a postcode. The server reads the basket from the session cart, so a test would need `POST /carts`, which writes to the dev database. I used QA's 16 existing dev orders instead. |
| The app and website screens on a device | Code only. I did not see the screens. |
| The per-order Standard price for the premium estimate | §5.3 uses averages. |
| Option-level lead times in the catalogue | Taken from the badge doc (production), not measured again. |

---

## Appendix A: queries used

All on dev MySQL `website_test`, read-only. Dev order ids above about 60,299 are test orders.

```sql
-- Delivery types and their texts
SELECT id,name,type,days,enabled,collection,enforce_whitelist,sort_order,updated_at
  FROM delivery_types WHERE is_deleted=0 ORDER BY collection,sort_order,id;
SELECT id,name,subtitle,details,days FROM delivery_types WHERE id IN (1,5,10,11,12,17);

-- Where lead time lives (only on order/cart lines)
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE FROM information_schema.COLUMNS
 WHERE TABLE_SCHEMA='website_test' AND (COLUMN_NAME LIKE '%lead%' OR COLUMN_NAME LIKE '%stock%'
   OR TABLE_NAME IN ('deliveries','delivery_types'));

-- Where the production copy ends
SELECT DATE(created_at) d, COUNT(*), MIN(id), MAX(id) FROM orders
 WHERE created_at BETWEEN '2026-06-15' AND '2026-07-10' GROUP BY d ORDER BY d;

-- §5.2 orders by type with a lead-time line (production copy)
WITH ol AS (
  SELECT o.id order_id, o.created_at, o.status, d.delivery_type_fk fk,
         MAX(COALESCE(p.leadTimeDuration,0)) max_lt, MAX(COALESCE(p.leadTime,0)) any_flag, COUNT(p.id) n_lines
  FROM orders o JOIN deliveries d ON d.id=o.delivery_A
  LEFT JOIN products p ON p.delivery_ID=d.id
  WHERE o.created_at >= '2023-04-03' AND o.created_at < '2026-06-24 12:28:00'
  GROUP BY o.id
)
SELECT COALESCE(dt.name, CONCAT('fk=',ol.fk)) dtype, dt.days, COUNT(*) orders, SUM(n_lines>0) with_lines,
  SUM(max_lt>0) with_lead_time, SUM(max_lt>=10) lt_ge10, MAX(max_lt) max_lt
FROM ol LEFT JOIN delivery_types dt ON dt.id=ol.fk GROUP BY dtype, dt.days ORDER BY orders DESC;

-- §5.2 lead time longer than the service, all statuses (with and without the leadTime flag)
WITH ol AS (
  SELECT o.id order_id, o.created_at ocreated, d.delivery_type_fk fk, d.price dprice,
         MAX(CASE WHEN p.leadTime=1 THEN COALESCE(p.leadTimeDuration,0) ELSE 0 END) lt_flagged,
         MAX(COALESCE(p.leadTimeDuration,0)) lt_any
  FROM orders o JOIN deliveries d ON d.id=o.delivery_A JOIN products p ON p.delivery_ID=d.id
  WHERE o.created_at >= '2023-04-03' AND o.created_at < '2026-06-24 12:28:00'
    AND d.delivery_type_fk IN (5,11,12,1,17,27)
  GROUP BY o.id
)
SELECT dt.name, dt.days, SUM(lt_any > dt.days) lead_longer_than_service,
  SUM(lt_flagged > dt.days) same_but_flag_on, ROUND(SUM(CASE WHEN lt_any > dt.days THEN dprice END),2) delivery_paid
FROM ol JOIN delivery_types dt ON dt.id=ol.fk GROUP BY dt.name, dt.days ORDER BY dt.days;

-- §5.2 completed only, with money
WITH ol AS (
  SELECT o.id, o.status, d.delivery_type_fk fk, d.price dprice, MAX(COALESCE(p.leadTimeDuration,0)) lt
  FROM orders o JOIN deliveries d ON d.id=o.delivery_A JOIN products p ON p.delivery_ID=d.id
  WHERE o.created_at >= '2023-04-03' AND o.created_at < '2026-06-24 12:28:00' AND d.delivery_type_fk IN (5,11,12)
  GROUP BY o.id
)
SELECT fk, COUNT(*), ROUND(SUM(dprice),2), MIN(lt), MAX(lt), SUM(lt>=10)
FROM ol WHERE status=1 AND lt > CASE fk WHEN 5 THEN 1 WHEN 11 THEN 0 ELSE 3 END GROUP BY fk;

-- §5.2 by year, and §5.3 average delivery price
WITH ol AS (
  SELECT o.id, o.created_at oc, d.delivery_type_fk fk, d.price dprice, MAX(COALESCE(p.leadTimeDuration,0)) lt
  FROM orders o JOIN deliveries d ON d.id=o.delivery_A JOIN products p ON p.delivery_ID=d.id
  WHERE o.created_at >= '2023-04-03' AND o.created_at < '2026-06-24 12:28:00' AND d.delivery_type_fk IN (1,5,11,12)
  GROUP BY o.id
)
SELECT fk, YEAR(oc), COUNT(*), SUM(lt > CASE fk WHEN 5 THEN 1 WHEN 11 THEN 0 WHEN 12 THEN 3 ELSE 5 END)
  FROM ol WHERE fk IN (5,11) GROUP BY fk, YEAR(oc);
-- (same CTE) SELECT fk, ROUND(AVG(dprice),2), COUNT(*) FROM ol GROUP BY fk;

-- Status of the 84 Next Day orders (status ids from order_statuses: 1 completed, 2 pending, 4 cancelled, 5 test, 6 closed)
SELECT o.status, COUNT(*) FROM orders o JOIN deliveries d ON d.id=o.delivery_A
 WHERE o.created_at >= '2023-04-03' AND o.created_at < '2026-06-24 12:28:00' AND d.delivery_type_fk=5
   AND EXISTS (SELECT 1 FROM products p WHERE p.delivery_ID=d.id AND p.leadTimeDuration>1)
 GROUP BY o.status;

-- §5.4 dev test orders
SELECT o.id, o.created_at, dt.type, d.price, d.selected_date, MAX(p.leadTimeDuration) max_lt
FROM orders o JOIN deliveries d ON d.id=o.delivery_A JOIN delivery_types dt ON dt.id=d.delivery_type_fk
JOIN products p ON p.delivery_ID=d.id
WHERE o.created_at >= '2026-06-24 12:28:00' AND dt.type IN ('NEXT_DAY','SAME_DAY','EXPRESS')
GROUP BY o.id HAVING max_lt > 0 ORDER BY o.id;
```

Catalogue, dev GraphQL (read-only, no auth needed for this query):

```bash
curl -s -X POST -H 'content-type: application/json' \
  --data '{"query":"{ allProducts { id sku status visible productType leadTime leadTimeDuration dateDueBackInStock inStock updated } }"}' \
  https://cdn.dev.firstfence.co.uk/graphql
# then count: status=PUBLISHED, visible != false, productType not in (GROUP, POA), leadTimeDuration > 0
```

Dev website build date:

```bash
curl -sI https://dev.firstfence.co.uk/page-data/app-data.json | grep -i last-modified
```

---

## Code reference index

**website-api**
- `app/Controllers/Http/DeliveryController.js:472` lead time found; `:481-494` type list;
  `:496-506` dates; `:545-593` installation / weight / blocked-date filters; `:860` cut-offs
- `app/Services/delivery/products.js:165-175` `findLeadTimeDays` (ignores the `leadTime` flag)
- `app/Services/delivery/dates.js:38-45, 76-78` collection dates; `:247-255, 323-325` delivery dates
- `app/Services/delivery/deliveryTypes.js:22-165` `getDeliveryAndCollectionTypes`
- `app/Services/delivery/delivery-cutoffs.js:85-121` 10am / 3pm rules
- `app/Services/delivery/installation.js:8-18` Specific Day only with installation
- `app/Services/delivery/blacklist.js:8-20` Glasgow rule; `blacklist-rows.js:21-39` blacklist rows
- `app/Middleware/CartProduct.js:23-29, 60-61` basket comes from the MySQL cart
- `app/Controllers/Http/CartController.js:397-398, 432-433` lead time saved as sent
- `app/Controllers/Http/OrderController.js:570-581, 594-598` delivery saved from the client
- `app/Utils/expected-delivery-date.js:271-282, 404-431` app expected date (FIR-266)
- `app/Utils/salesforce-quotes.js:85-139` `getDueDate`; `:329-339` lead time for Salesforce
- `app/Controllers/Http/SalesforceQuoteWizardController.js:304, 348, 605, 649` `Due_Date__c`,
  `Lead_Time_In_Days__c`
- `app/Controllers/Http/DeliveryTypeController.js:11-45, 62-110` admin save of delivery types

**gatsby-website**
- `src/pages/checkout/delivery/index.js:126-129` call; `:141-143` filter; `:147-148` pre-select
- `src/features/checkout/service-page/delivery-options/delivery-options.js:29-42` list
- `src/features/checkout/service-page/delivery-option/delivery-option.js:17-38` option card
- `src/features/checkout/service-page/calendar/date-tiles.js:31, 58` lead-time calendar tiles
- `src/pages/checkout/contact-details/index.js:214-215, 233-235` order payload
- `src/features/checkout/sidebar/delivery/delivery.js:39-44` 10pm note
- `src/utils/product/lead-time.js:34-73`, `src/models/Product.js:688, 759-760` line lead time
- `src/utils/portal/order-delivery.js:99-133` portal date after the order

**ff-uk-mobile**
- `src/features/delivery/services/deliveryApi.ts:34-35` call
- `src/features/delivery/utils/deliveryEstimate.ts:15-22, 79-101` estimate with lead time
- `src/features/delivery/utils/availableDates.ts:90` lead-time calendar dates
- `src/features/checkout/utils/draftDeliveryDate.ts:33` no date for Next Day
- `src/features/basket/utils/cartRow.ts:26-27`, `src/core/pricing/configurator.ts:244-248` line lead time

**admin-website-v2**
- `src/pages/content/delivery/delivery-types/delivery-type-page/` (`general-fields/general-fields.jsx:47-160`)
  delivery type form, no lead-time field
