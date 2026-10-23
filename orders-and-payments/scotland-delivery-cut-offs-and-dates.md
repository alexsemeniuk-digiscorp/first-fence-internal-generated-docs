# Is there a Scotland cut-off for delivery times or dates?

QA asked: *"Is there a cut-off for Scotland time/dates in the project?"* This doc answers that
question. It also lists every rule the system has for Scottish addresses, and the one Scotland
rule the business asked for that was never built.

**Data provenance:** dev MySQL (`website_test` @ `54.171.181.199`), read-only, on **2026-10-23**
(about 11:30 to 11:50 UTC). Dev GraphQL (`https://cdn.dev.firstfence.co.uk/graphql`): read-only
`allCMSPages` (63 pages) and `allHolidayMessages` (3 messages) queries. Dev REST API: 11 read-only
`POST /delivery` quotes at **12:35 BST**, using an existing anonymous dev cart (id 4870158, one
in-stock line). `/delivery` does no writes, and the `carts` table had the same max id (4870244)
before and after. The dev Mongo port 27017 did not accept connections, so CMS data came from the
GraphQL only. Linear was searched for "Scotland", "cut-off" and "cutoff". Production was not
touched. Code read at:

| Repo | Ref | Date | Local refs fetched |
|---|---|---|---|
| website-api | `mobile-app` `8be170f`; `origin/master` `d27b5cc`; `origin/dev-docker` `a183a69` | 2026-10-22 | 2026-10-22 17:46 |
| gatsby-website | `master` `b8b24861e`; `origin/test-site` `b806887ee` | 2026-10-14 | 2026-10-15 17:28 |
| ff-uk-mobile | `dev` `94e2b87` | 2026-10-21 | 2026-10-22 17:07 |
| admin-website-v2 | `mobile-app` `5f42d3c` | 2026-10-22 | 2026-10-22 17:35 |
| cdn-graphql-v2 | `mobile-app` `d53484b` | 2026-10-21 | 2026-10-22 11:33 |

"Real orders" below means completed orders (`orders.status = 1`) in the production copy on dev,
that is, created up to **2026-06-24 12:28 UTC**. Later rows are dev test orders. The shipping
postcode comes from `orders.person_ID → people.shipping_address → addresses.post_code`.

Companion docs: [`next-day-delivery-with-lead-time.md`](./next-day-delivery-with-lead-time.md)
(the 10am / 3pm / 10pm cut-offs in more detail),
[`delivery-types-and-options.md`](./delivery-types-and-options.md) (all delivery types and fields).

---

## TL;DR

**No. There is no Scotland cut-off anywhere in the system. This is true for the time of day and
for dates.** Every order cut-off is the same for every UK postcode:

- The API removes Same Day at 10:00.
- The API reprices or removes Next Day at 15:00.
- The website, the app and the order email say 10pm for Next Day.

None of these looks at the postcode.

**But the business did ask for one.** On 2026-08-07 Charlotte McCairn said Next Day can be ordered
up to 10pm **"(excluding Scotland)"**. The developer moved the cut-off to 10pm and chose not to
build the Scotland part. The commit message says the caveat would reach customers through the Next
Day description text in admin (api `150ef7e`, web `8ca1a50c8`, both on `master` the same day).
On dev, that text still says **"Available on orders placed before 2.30pm"** and does not mention
Scotland. No CMS page, holiday message, email or app screen mentions Scotland either. Nobody wrote
down what the Scotland cut-off should be.

Dates work the same way. Blocked dates (bank holidays, stock takes, Christmas) are attached to
**delivery types and collection depots**, never to a region. So a Scottish bank holiday can be
blocked for **collection at the Glasgow depot**. Admins have done this 5 times since 2023. It
cannot be blocked for **deliveries to Scottish addresses** without blocking it for the whole UK.

What the system **does** do for Scotland (details in §5):

| Rule | Where it lives | Time or date effect? |
|---|---|---|
| Higher minimum prices per postcode area. All delivery types blocked for the Outer Hebrides (HS), Shetland (ZE), Cumbrae (KA28) and PA77 (Tiree). | data: `delivery_post_codes` | No |
| Stripe mesh kits and residential gates: Standard only in 13 Scottish areas | code: `postcode-delivery-rules.js` | No |
| Scottish bank holidays blocked by hand for Glasgow collection | data: `delivery_blocked_dates` | Yes, Glasgow collection only |
| **25 island districts lose Next Day at 15:00**, because Same Day is blocked there | side effect of data + `delivery-cutoffs.js` | **Yes, by accident** |

**For QA:** there is nothing Scotland-specific to test for cut-offs. Use the same 10am / 3pm /
10pm checks as for England (§2). If the business still wants "10pm, excluding Scotland", it is a
new feature that needs a business answer first (§8).

---

## 1. What the question can mean

Scotland uses the same clock as England (GMT in winter, BST in summer). So "Scotland time" cannot
differ by time zone. It can only differ by a rule. There are two possible readings:

| Reading | Short answer | Section |
|---|---|---|
| A different **order cut-off time** for Scottish addresses (for example, Next Day by 2pm in Scotland instead of 10pm) | Not built. The business asked for it on 2026-08-07, and the developer decided to put it in text only. The text was never written on dev. | §2, §3 |
| Different **dates** for Scotland (Scottish bank holidays: 2 January, the first Monday in August, St Andrew's Day) | Only for collection at the Glasgow depot, entered by hand. Deliveries to Scotland use the same calendar as England. | §4 |

---

## 2. Every cut-off in the system, and whether it checks the postcode

| Where | Rule | Clock used | Checks postcode? | Code |
|---|---|---|---|---|
| API, `/delivery` | From 10:00, Same Day is removed | server clock (`moment()`) | No | `website-api/app/Services/delivery/delivery-cutoffs.js:12`, `:115-120` |
| API, `/delivery` | From 15:00 (when tomorrow is a weekday), Next Day loses AM/PM and gets the Same Day price. If there is no Same Day price, Next Day is removed. | server clock | No | `delivery-cutoffs.js:15`, `:97-113`; called at `DeliveryController.js:860` |
| API, `/delivery` | 12:00 to 22:00: `next_day_increase` price modifier on Next Day and Same Day | server clock | No | `app/Services/delivery/deliveryTypes.js:171-212` |
| Website checkout sidebar | After 22:00: "Your order will be processed tomorrow due to our Next Day Delivery cut-off time of 10pm" | **browser clock** | No | `gatsby-website/src/features/checkout/sidebar/delivery/delivery.js:39-44`, `:74-77` |
| Website header | Countdown to 22:00 | **browser clock** | No | `src/layout/header/delivery-banner/delivery-banner.js:9` |
| Website category pages | schema.org `cutoffTime: "22:00"` | (static) | No | `src/templates/category/category.js:431` |
| App checkout and basket | Next Day 22:00, Same Day 10:00. Warning text and "Why this date?" row "Cut-off time: 10pm" | UK clock (`src/lib/ukTime.ts`) | No | `ff-uk-mobile/src/features/delivery/utils/deliveryEstimate.ts:31-37`; `src/features/checkout/screens/CheckoutScreen.tsx:810-811`; `src/ui/components/WhyDateSheet.tsx:63-64` |
| API, app expected date (orders list) | NEXT_DAY placed from 22:00 UK, SAME_DAY from 10:00 UK, count from the next day | UK clock | No | `website-api/app/Utils/expected-delivery-date.js:72-73`, `:299-301` (on `dev-docker`, **not on `master`**) |
| Order confirmation email | "Next Day (order before 10pm)" | (static) | No | `app/Utils/shipping-services.js:58`, shown by `resources/views/emails/order/delivery-info.edge` |
| Admin text, `delivery_types.details` (dev) | Next Day: "Available on orders placed before 2.30pm, delivery on working days only. ..." Same Day: "Available only on orders placed before 10am, subject to stock availability." | (static) | No | row id 5, `updated_at` 2026-03-02 07:04; row id 11, `updated_at` 2025-02-28 |

The only Scotland code in website-api is the product rule in §5.2. `git grep -il scotland` on
every remote branch finds the same three files and nothing else:
`DeliveryController.js`, `postcode-delivery-rules.js` and `delivery-constants.js`. gatsby-website
has no Scotland logic. Its only Scotland text is on the Glasgow depot page. ff-uk-mobile has no
Scotland logic, and its only Scotland text is in the BreezeGuard wind table.

**For QA testing from Ukraine.** The website's 10pm checks use the **browser** clock. In Kyiv the
browser is 2 hours ahead of the UK, so the 10pm warning appears at 20:00 UK time. The app uses UK
time whatever the phone's time zone is. The API uses its server clock (§10).

---

## 3. The "(excluding Scotland)" decision: dates

| Date | Where | What happened |
|---|---|---|
| 2025-07-30 | web `41cfee55e` | Checkout sidebar warning added, at **2:30pm** |
| 2026-03-02 07:04 UTC | dev data | Next Day `details` last edited. It says "before 2.30pm" today. |
| 2026-06-24 12:28 UTC | dev data | End of the production copy. Admin edits made on prod after this are not on dev. |
| **2026-08-07** | api `150ef7e`, web `8ca1a50c8`, both committed **directly on `master`** | "Charlotte McCairn: next day deliveries can be ordered up to 10pm." Both messages end with: *"The "(excluding Scotland)" caveat is not modelled in code by decision; it reaches customers through the delivery type description in admin."* Changed: website sidebar, header countdown and schema.org to 22:00; API price modifier window to 22:00; email text to "order before 10pm". |
| 2026-09-17 | api `18ae389` (sc-1539), merged to `master` in `ca380e9` the same day | 10am Same Day removal and 3pm Next Day reprice/removal. "Charlotte, via Joe". No postcode check, and Scotland is not mentioned. |
| 2026-09-26 to 09-28 | app `10077b3`, `880c22f` | App dates use the 10pm / 10am cut-offs. No Scotland rule. |

Where the caveat was supposed to appear: `delivery_types.details` for NEXT_DAY. Both clients show
that field under the option:

- website: `gatsby-website/src/features/checkout/service-page/delivery-option/delivery-option.js:37-38`
- app: `ff-uk-mobile/src/features/checkout/screens/CheckoutScreen.tsx:752-758`

What dev holds today:

| Place a customer could read a Scotland cut-off | On dev |
|---|---|
| Next Day `details` | "before 2.30pm". No Scotland. |
| 63 CMS pages (GraphQL `allCMSPages`) | Scotland appears only in depot addresses and the Glasgow depot page. No cut-off text. |
| 3 holiday messages (GraphQL `allHolidayMessages`) | Christmas only |
| Order email, app, website | Hard-coded "10pm", with no exception |
| Linear | No issue mentions a Scotland cut-off |

So, on dev, **nothing tells a Scottish customer about a different cut-off**. Whether prod's Next
Day text was changed after 2026-08-07 is not visible from dev (query Q1 in §10).

Two more problems, even if the text gets written:

1. **Nobody wrote down the Scotland cut-off.** The commit says "excluding Scotland" only. It does
   not say what the Scotland time is, or what happens after it (reprice like the 3pm rule, or
   remove Next Day).
2. **The text would contradict 6 hard-coded "10pm" messages:** the website warning and countdown,
   the app warning and "Why this date?" row, the order email, and schema.org. It would also be
   shown to every English customer, because one `details` text serves all postcodes.

---

## 4. Dates: Scottish bank holidays

### 4.1 How blocked dates work

- Table `delivery_blocked_dates` (57 rows on dev). Each row links to delivery types through
  `delivery_date_type_pivot` (FK `ON DELETE CASCADE` on both sides). The collection depots are
  delivery types too (`collection = 1`; Glasgow is `delivery_types.id = 8`, type `GLASGOW`).
- **There is no region, postcode or country column.** The admin form
  (`admin-website-v2/src/pages/content/delivery/delivery-block-dates/add-date-modal/add-date-modal.jsx:115-158`)
  has Display Name, Details, Start Date, End Date, Reason, Action Taken and a delivery-type
  checklist. Nothing else.
- Effect on **delivery** types: a type is removed only when **today** is blocked for it
  (`DeliveryController.js:581-594`). Specific Day's calendar greys out its own blocked dates
  (`app/Services/delivery/dates.js:208-245`). The day Next Day lands on is **not** checked against
  blocked dates. This is the same for England and Scotland.
- Effect on **collection**: the website and the app grey out the chosen depot's blocked dates
  (`gatsby-website/src/pages/checkout/delivery/index.js:160-161`, `:235-236`;
  `ff-uk-mobile/src/features/delivery/utils/deliveryQuote.ts:151`).

So the only way to give Scotland its own dates today is to block them for the **GLASGOW**
collection type. That is what admins have done.

### 4.2 Glasgow-only blocked dates (dev, production copy)

| id | Dates | Display name | Created (UTC) |
|---|---|---|---|
| 10 | 2023-01-03 | "Glasgow Bank Holiday" | 2022-12-23 10:43 |
| 17 | 2023-09-25 | "Bank holiday" | 2023-09-20 12:56 |
| 21 | 2024-01-02 | "Scotland Bank Holiday" | 2023-12-15 10:11 |
| 33 | 2025-07-08 to 2025-07-09 | "Bank Holiday" | 2024-11-28 11:33 |
| 53 | 2025-08-01 | "Collections Unavailable" | 2025-08-01 12:44 |

All other rows block England and Scotland together.

### 4.3 Scottish bank holidays compared with what is blocked for Glasgow

Scottish bank holidays differ from England's in four ways. Scotland has 2 January, has its summer
holiday on the **first** Monday of August, and has St Andrew's Day (30 November). It does **not**
have Easter Monday or the **last** Monday of August.

| Day | Scotland | England | Blocked for GLASGOW on dev? |
|---|---|---|---|
| 2025-08-04 (first Mon Aug) | yes | no | **No** |
| 2025-08-25 (last Mon Aug) | no | yes | Yes (row 42) |
| 2025-12-01 (St Andrew's, moved from Sunday) | yes | no | **No** |
| 2026-01-02 | yes | no | Yes, inside Christmas row 43 (to 2026-01-04) |
| 2026-04-06 (Easter Monday) | no | yes | Yes (row 59, 2026-04-03 to 04-06) |
| 2026-08-03 (first Mon Aug) | yes | no | **No** |
| 2026-08-31 (last Mon Aug) | no | yes | Yes (row 65) |
| **2026-11-30 (St Andrew's)** | yes | no | **No** (next upcoming) |
| **2027-01-04 (2 Jan, moved from Saturday)** | yes | no | **No**. Christmas row 67 ends 2027-01-03. |

From the history it looks like Glasgow follows the company (English) calendar plus 2 January. That
is an inference. Whether the Glasgow depot really opens on St Andrew's Day and the first Monday
in August is a business question.

The dev table gets no new rows after the copy: `AUTO_INCREMENT` is 72 and the highest id is 71.
Prod may have rows added after 2026-06-24 (query Q2).

### 4.4 Dead code that looks like Scotland dates (do not test it)

| Code | Status |
|---|---|
| `blockCollectionDates` in `app/Services/delivery/blockDates.js:16-59`: hard-coded 2022 dates, with an extra `2022-09-26` for Glasgow | **No callers.** `grep -rn blockCollectionDates app start` finds only the definition. |
| `BANK_HOLIDAY_ARRAY` in `app/Utils/delivery-constants.js:1-48`: English holidays to 2022-12-27 | Exported but **imported nowhere** |
| Collection blocked-date check in `getCollectionDates`, `app/Services/delivery/dates.js:62-69` | Commented out in `80fa51c` (2025-08-01). The clients apply depot dates instead (§4.1). |

### 4.5 Your code: `DeliveryCalendarService` (dev-docker only)

`app/Services/DeliveryCalendarService.js` builds the app's expected date from the **union of all
`BLOCKED` rows**, whatever their type. So Glasgow-only Scottish dates also move English orders'
expected dates, and English-only dates move Scottish ones. The class comment explains this
choice and says the date can only come out later, never earlier. No Glasgow-only row on dev is in
the future, so there is no effect today. The file is not on `master`.

---

## 5. What the system does for Scottish addresses

### 5.1 Prices and blocks by postcode (data, `delivery_post_codes`)

Matching is by **longest prefix**, for each delivery type separately (`deliveryTypes.js:54-77`).
Minimum prices on dev for the main types:

| Area | Standard | Next Day | Same Day | Express |
|---|---|---|---|---|
| G, ML, FK | £88 | £93 (G) / £115 | £225 | £93 |
| EH, KY | £120 | £214 | £296 | £154 |
| DD, DG, PH, TD | £176 | £280 | £390 | £181 |
| AB | £220 | £357 | £467 | £225 |
| IV | £445 | £472 | £582 | £450 |
| KW | £582 | £610 | £720 | £588 |
| PA (area default) | £176 | £115 | £225 | £252 |
| PA20-PA38 (Argyll, Bute) | £252-£445 | £280-£500 | £390-£582 | £258-£450 |
| PA41, PA43-PA49, PA60-PA76 (Islay, Jura, Mull, ...) | £582 | £912 | **blocked** | £588 |
| PA42 | **blocked** | **blocked** | **blocked** | £588 |
| HS, ZE, KA28, PA77 | **blocked** | **blocked** | **blocked** | **blocked** |

The island block rows were created on 2021-09-09 (PA Same Day) and 2022-08-11 / 2022-10-07 (HS,
ZE). These are production rows.

Two dev-only details matter for QA:

- **Ayrshire (KA) Standard is blocked on dev only.** Row 227 (`KA`, STANDARD) was set to
  `blocked = 1` on **2026-09-10 08:13 UTC**, and row 3156 (`KA27`) was edited at 08:19. Both
  edits were made after the copy, on dev. Dev cannot show what row 227 held before the edit
  (query Q3 checks prod). On dev, a KA postcode gets no Standard delivery. Do not assume prod
  does the same.
- **Economy has no HS or ZE row.** A dev quote for `HS1 2RF` returns only `ECONOMY £161`. One real
  completed Economy order went to HS7 (order 36716, 2024-04-26, £322.90).

### 5.2 The Scotland product rule (code)

`applyScotlandBlacklistRestriction` (`app/Services/delivery/postcode-delivery-rules.js:84-94`,
called at `DeliveryController.js:850`) keeps **only Standard** when both of these are true:

- the postcode area is in `SCOTLAND_POSTCODES` (`app/Utils/delivery-constants.js:207-221`: AB, DG,
  ML, G, IV, DD, KY, FK, PH, KW, EH, PA, TD);
- the basket has a SKU in `SCOTLAND_BLACKLISTED_PRODUCTS` (`:223-228`), or a residential gate.

The 4 SKUs are the Stripe Mesh Security Fencing kits 1.8m, 2.0m and 2.4m (WEB-00132, WEB-00134,
WEB-00136) and Stripe Mesh Sheets (WEB-01348).

| Date | Commit | Change |
|---|---|---|
| 2025-12-02 | `d30b75a` (directly on `master`) | Rule added. It matched by first letters, so English DE, E and M postcodes were also caught. |
| 2026-07-15 | `38233fc` | Residential gates added |
| 2026-07-16 | `b3338fb` (merged `69fcda6`) | Exact area match (`postcodeArea`, `:25-29`) |

It is about products, not time. Since 2025-12-02, 3 real completed stripe-mesh orders went to
Scotland, and all 3 were Standard (orders 55583, 58378, 58541). The rule works. Gaps in the list:

| Gap | Real completed orders (all years) | Effect |
|---|---|---|
| **KA** (Ayrshire) missing | 200 to KA; 0 stripe-mesh orders to KA since the rule | Ayrshire gets stripe mesh by any service |
| HS, ZE missing | 19 HS, 3 ZE (mostly collections) | No effect: every service except Economy is blocked there anyway |
| TD15 (Berwick-upon-Tweed) and part of TD12 are in **England** | 15 to TD15 | Berwick is treated as Scotland |

### 5.3 The accidental Scotland cut-off: islands lose Next Day at 15:00

From 15:00, `applyDeliveryCutoffs` copies the Same Day price into Next Day. If the basket has no
Same Day option, it **removes Next Day** (`delivery-cutoffs.js:102-112`). Same Day is blocked by
postcode in these 25 Scottish districts while Next Day is open:

> PA41, PA43, PA44, PA45, PA46, PA47, PA48, PA49, PA60, PA61, PA62, PA63, PA64, PA65, PA66, PA67,
> PA68, PA69, PA70, PA71, PA72, PA73, PA74, PA75, PA76

So for these islands, **Next Day is offered until 14:59 (at £912 minimum) and disappears from
15:00**. On the mainland, Next Day stays after 15:00 at the Same Day price (for example G: £225
minimum instead of £93). This is the only time-of-day difference between Scottish addresses and the
rest of the UK. It comes from the 2021 island data meeting the 2026 3pm rule. Nobody designed it
for Scotland. The sc-1539 commit lists it under "19 postcode prefixes" that lose Next Day after
3pm. On today's dev data it is 26 prefixes: these 25 plus EC3.

### 5.4 Postcodes typed without a space get the wrong district

The API only trims the postcode (`normalizePostcode`, `postcode-delivery-rules.js:14-17`, called at
`DeliveryController.js:399`). It keeps an inner space, and it does not add a missing one. The
website sends the saved address postcode as typed (`gatsby-website/src/pages/checkout/delivery/index.js:100`,
`:127-128`). The prefix match then reads the inward code as part of the district. Dev quotes,
same cart, at 12:35 BST:

| Postcode sent | Standard | Next Day | Express | Matched prefix |
|---|---|---|---|---|
| `PA4 9EN` (Renfrew) | £176 | £160 | £252 | PA |
| `PA49EN` (same address) | **£582** | **£912** | **£588** | PA49 (Islay) |
| `PA7 5AA` | £176 | £160 | £252 | PA |
| `PA75AA` | **£582** | **£912** | **£588** | PA75 (Mull) |
| `PA6 7AA` | £176 | £160 | £252 | PA |
| `PA67AA` | **£582** | **£912** | **£588** | PA67 (Mull) |
| `G73 1AF` and `G731AF` | £88 | £160 | £93 | G (G has no district rows, so the space does not matter) |

From 15:00 the unspaced versions also lose Next Day (§5.3). Real impact so far is small:

- 49 of 2,652 completed Scottish orders were saved without a space.
- 2 of those match a different rule: order 790 (`PA49EN`) and order 6620 (`PA26RT`). Both were
  collections.
- Across the UK, 11 completed orders match a different rule (also GU, PE, PO).

Customers who saw £912 and left do not appear in the data. The fix belongs to website-api: match on
the outward code only. All 3,537 prefixes are outward codes or shorter (no spaces, at most 4
characters), so this is safe for full postcodes. Trap: drop the last 3 characters only when the
string is a full postcode (`^[A-Z]{1,2}\d[A-Z\d]?\s*\d[A-Z]{2}$`). Otherwise a caller that sends
only the outward code (`PA4`) would be cut to nothing.

---

## 6. Numbers

Real completed orders, 2019-11-05 to 2026-06-24: **46,562**. Of these, **2,652 (5.7%)** went to
the 16 Scottish postcode areas.

| Service | Scotland | All UK |
|---|---|---|
| Standard | 979 | 24,445 |
| Collection | 890 | 8,184 |
| Express | 290 | 5,246 |
| **Next Day** | **159** (£36,201.70 delivery charged) | 3,604 |
| Economy | 120 | 1,591 |
| DPD | 104 | 1,404 |
| Specific Day | 74 | 1,721 |
| **Same Day** | **22** (£5,375.47) | 152 |

By area: G 716, EH 402, ML 269, KY 207, KA 200, AB 152, PA 149, FK 141, DD 102, PH 94, DG 88,
TD 52, IV 50, HS 19, KW 8, ZE 3.

When Scottish Next Day orders were placed (UK time). No time cut-off removed options before
2026-09-17, so these numbers show what customers want, not a fault:

| Placed at or after | Scottish Next Day orders | Share |
|---|---|---|
| 10:00 | 130 of 159 | 82% |
| 14:30 | 46 of 159 | 29% |
| 15:00 | 42 of 159 | **26.4%** (rest of UK: 821 of 3,445 = 23.8%) |
| 22:00 | 2 of 159 | 1% |
| Since 2025-01-01, at or after 15:00 | 13 of 43 | 30% |

If the business sets a Scotland Next Day cut-off at 2.30pm or 3pm, it would affect about one in
four Scottish Next Day orders. That is about 9 orders a year (13 in the 18 months from January
2025). Scottish customers order after 15:00 only a little more often than the rest of the UK.

Same Day to Scotland: 18 of 22 orders were placed at or after 10:00. That was before the 10am rule
existed (2026-09-17). It is the same for all of the UK.

---

## 7. Is this just dev?

| Question | Answer |
|---|---|
| Can a deploy cause it on prod? | No deploy is needed. Prod already has the same code. `150ef7e` and `d30b75a` are first-parent commits on `master`, and `18ae389` and `b3338fb` are merged. Web `8ca1a50c8` is on gatsby `master` and `test-site`. CI deploys `master` (`website-api/.gitlab-ci.yml`). |
| Can it happen again after the fixes in the repo? | There is no fix. No branch on any repo has a Scotland cut-off. |
| Is prod already in that state? | Code: yes, no Scotland cut-off. Data: not verified. Prod may have a new Next Day `details` text, or new Glasgow dates after 2026-06-24 (queries Q1, Q2). |

---

## 8. If the business wants a Scotland cut-off

First the business has to answer three questions (Charlotte McCairn made the 10pm decision):

1. What is the Scotland Next Day cut-off time?
2. After it, should Next Day be repriced (like the 3pm rule) or removed?
3. Does the Glasgow depot close on St Andrew's Day, 2 January and the first Monday of August? And
   should Scottish deliveries stop on those days?

| Option | What changes | Traps |
|---|---|---|
| **A. Text only** (what was decided on 2026-08-07) | Admin: edit the NEXT_DAY `details` text. No code. | One text serves all postcodes, so English customers read the Scotland caveat too. Six hard-coded "10pm" messages still contradict it (§3). The current text says "2.30pm", which is already wrong for England. Easy to undo. |
| **B. API rule** | website-api: in `applyDeliveryCutoffs`, use an earlier hour when `isScotlandPostcode(postcode)`. Return the cut-off on the NEXT_DAY option, for example `cutoff: "14:00"`. Web and app: read that field instead of hard-coding 22:00. Your `expected-delivery-date.js:72-73` needs the postcode too. | Fix `SCOTLAND_POSTCODES` first (add KA, HS, ZE; decide on TD15), or Ayrshire keeps the English rule. The API uses the server clock (§10). Changes are needed in 3 repos, or the messages disagree. Easy to undo: one constant, or a `delivery_variables` row. |
| **C. Scottish dates for collection** | Admin only: add blocked dates for GLASGOW, for example 2026-11-30 and 2027-01-04. | Not on master: `DeliveryCalendarService` would also move English expected dates (§4.5). |
| **D. Scottish dates for delivery** | Needs a region on `delivery_blocked_dates`, or a postcode check next to `DeliveryController.js:581-594`. The admin form needs a new field. | A big change for a few days a year. The day Next Day lands on is not checked against blocked dates for anyone today (§4.1). Fix that first, or D does nothing for Next Day. |

**Recommendation:** get answers to questions 1-3. Until then, change nothing. If the answer is "yes,
enforce it", choose B and fix the area list in the same change. Separately, fix §5.4 (postcode
without a space). It is a small, safe website-api change and does not need a business decision.

---

## 9. Reconciliation with existing docs

| Doc | Status |
|---|---|
| [`next-day-delivery-with-lead-time.md`](./next-day-delivery-with-lead-time.md) §6 | **Was wrong in one row. Fixed in this commit.** It said web `41cfee55e` (2025-07-30) added a "10pm" note. That commit added a **2:30pm** note. 10pm came on 2026-08-07 (`8ca1a50c8`). §11 item 1 ("three different Next Day cut-off times") is still correct. It now notes the unbuilt Scotland caveat. |
| [`delivery-types-and-options.md`](./delivery-types-and-options.md) | Still correct. It does not cover cut-offs or Scotland. Its count of 3,537 postcode rules still matches dev. |
| `website-api/.generated_docs/App-Status-Values-Reference.md:466` | Correct for the code (22:00 / 10:00 UK, no Scotland). |
| Commit messages `150ef7e` / `8ca1a50c8` | They say the caveat "reaches customers through the delivery type description in admin". **On dev it does not.** The text still says 2.30pm. |
| `website-api/.generated_docs/FirstFence-Meetings.md` | No mention of a Scotland cut-off |

---

## 10. Not verified

- **Prod Next Day text and prod blocked dates.** Run these read-only queries on the production
  MySQL:

  ```sql
  -- Q1: does prod's Next Day text mention Scotland or 10pm?
  SELECT id, type, details, updated_at FROM delivery_types WHERE type IN ('NEXT_DAY','SAME_DAY');

  -- Q2: Glasgow / Scotland dates added after the dev copy
  SELECT b.id, b.start_date, b.end_date, b.display_name, b.action, b.created_at,
         GROUP_CONCAT(dt.type ORDER BY dt.id) AS types
  FROM delivery_blocked_dates b
  LEFT JOIN delivery_date_type_pivot p ON p.delivery_blocked_date_id = b.id
  LEFT JOIN delivery_types dt ON dt.id = p.delivery_type_id
  WHERE b.created_at > '2026-06-24 12:28:13' OR b.end_date >= '2026-10-23'
  GROUP BY b.id ORDER BY b.start_date;

  -- Q3: is Ayrshire Standard open on prod? (blocked on dev by a 2026-09-10 test edit)
  SELECT id, postcode_prefix, blocked, min_price, updated_at FROM delivery_post_codes WHERE id IN (227, 3156);
  ```

- **The API server's time zone.** `applyDeliveryCutoffs` uses `moment()`, which is the server's
  local time. The repo does not set `TZ` (`Dockerfile`; `.gitlab-ci.yml` deploys over SSH to
  `pm2`). If the host runs UTC, the 10:00 and 15:00 rules fire one hour late (11:00 and 16:00 UK)
  during BST. This does not matter from 2026-10-25 to 2027-03-28, when UK time equals UTC. To
  check, run on the dev and prod hosts: `pm2 env 0 | grep -i '^TZ'; date; timedatectl | grep 'Time zone'`.
- **What the Scotland cut-off should be.** It is not written anywhere. Only the business can say.
- **Whether the Glasgow depot opens on Scottish bank holidays.** The pattern in §4.3 is an
  inference from the data.
- **Customers who left because of the no-space price (§5.4).** This cannot be measured.
- **Which depot fulfils Scottish deliveries,** and whether that changes the real dispatch cut-off.
  SAP was not checked.
- **Single-letter prefix rows outside Scotland.** For example, `G` has a Same Day row, but `GU` and
  `GY` have none at area level. I did not check whether Guildford or Guernsey fall through to
  Glasgow's Same Day price. It is the same kind of bug as the one fixed in `b3338fb`, and is worth
  a separate look.
- The 7 hard-deleted dev `delivery_post_codes` rows: `AUTO_INCREMENT` is 3696, the highest id is
  3688. Not looked into.

---

## Appendix A. Queries

All against dev `website_test`, read-only, through
`docker exec -i firstfence-mysql mysql -h 54.171.181.199 -u mobile_dev website_test`.

```sql
-- Delivery types and their customer-facing text
SELECT id, name, type, days, collection, enabled, subtitle, details, updated_at
FROM delivery_types ORDER BY collection, sort_order;

-- Blocked dates with their delivery types
SELECT b.id, b.start_date, b.end_date, b.reason, b.display_name, b.action, b.created_at,
  (SELECT GROUP_CONCAT(dt.type ORDER BY dt.id) FROM delivery_date_type_pivot p
     JOIN delivery_types dt ON dt.id = p.delivery_type_id
   WHERE p.delivery_blocked_date_id = b.id) AS types
FROM delivery_blocked_dates b ORDER BY b.start_date;

-- Postcode rules for Scottish areas
SELECT pc.postcode_prefix,
  GROUP_CONCAT(CONCAT(dt.type, ':', IF(pc.blocked, 'BLOCKED', ROUND(pc.min_price))) ORDER BY dt.id SEPARATOR ' ') AS rules
FROM delivery_post_codes pc JOIN delivery_types dt ON dt.id = pc.delivery_type_id
WHERE dt.id IN (1,5,10,11,12,17)
  AND REGEXP_SUBSTR(UPPER(pc.postcode_prefix), '^[A-Z]+')
      IN ('AB','DD','DG','EH','FK','G','HS','IV','KA','KW','KY','ML','PA','PH','TD','ZE')
GROUP BY pc.postcode_prefix ORDER BY pc.postcode_prefix;

-- Dev-only postcode rule edits (after the production copy)
SELECT pc.id, pc.postcode_prefix, dt.type, pc.blocked, pc.min_price, pc.created_at, pc.updated_at
FROM delivery_post_codes pc JOIN delivery_types dt ON dt.id = pc.delivery_type_id
WHERE pc.updated_at > '2026-06-24 12:28:13' OR pc.created_at > '2026-06-24 12:28:13';

-- Every order with shipping postcode and service (exported, then counted in Python)
SELECT o.id, o.created_at, o.status, UPPER(TRIM(a.post_code)) AS pc,
  REGEXP_SUBSTR(UPPER(TRIM(a.post_code)), '^[A-Z]+') AS area,
  d.delivery_type, d.delivery_time_frame, d.price, d.selected_date
FROM orders o
JOIN people p ON p.id = o.person_ID
JOIN addresses a ON a.id = p.shipping_address
LEFT JOIN deliveries d ON d.id = o.delivery_A
WHERE o.created_at IS NOT NULL;

-- Stripe mesh orders to Scotland since the rule
SELECT o.id, o.created_at, UPPER(TRIM(a.post_code)) AS pc, d.delivery_time_frame, d.price,
  GROUP_CONCAT(DISTINCT pr.sku) AS skus
FROM orders o JOIN people pe ON pe.id = o.person_ID JOIN addresses a ON a.id = pe.shipping_address
JOIN deliveries d ON d.id = o.delivery_A JOIN products pr ON pr.delivery_ID = o.delivery_A
WHERE pr.sku IN ('WEB-00132','WEB-00134','WEB-00136','WEB-01348') AND o.status = 1
  AND REGEXP_SUBSTR(UPPER(TRIM(a.post_code)), '^[A-Z]+')
      IN ('AB','DD','DG','EH','FK','G','HS','IV','KA','KW','KY','ML','PA','PH','TD','ZE')
  AND o.created_at >= '2025-06-01'
GROUP BY o.id;
```

Python counting rules: the cut is `created_at <= 2026-06-24 12:28:13` and `status = 1`. UK time is
UTC + 1 hour from 01:00 UTC on the last Sunday of March to 01:00 UTC on the last Sunday of
October. The service is `delivery_time_frame` upper-cased with spaces changed to `_`:
`STANDARD_FREE` and `STANDARD_JANUARY_OFFER` count as STANDARD, and `EXPRESS_JANUARY_OFFER` as
EXPRESS. The longest-prefix match copies `deliveryTypes.js:58-71`.

Dev API quotes (§5.4), one per postcode:

```bash
curl -s https://api.dev.firstfence.co.uk/delivery -H 'content-type: application/json' \
  -H "x-cart-token: <sessionID of cart 4870158>" -d '{"postcode":"PA49EN"}'
```

Dev GraphQL:

```graphql
{ allCMSPages { id path title status updated html renderedHtml blocks } }
{ allHolidayMessages { id title message startDate endDate updatedAt } }
```

## Appendix B. Code index

**website-api**
- `app/Services/delivery/delivery-cutoffs.js:12,15` cut-off hours; `:85-121` `applyDeliveryCutoffs`
- `app/Services/delivery/postcode-delivery-rules.js:14-17` `normalizePostcode`; `:25-29`
  `postcodeArea`; `:43-47` `isScotlandPostcode`; `:84-94` Scotland restriction
- `app/Utils/delivery-constants.js:1-48` `BANK_HOLIDAY_ARRAY` (dead); `:207-221`
  `SCOTLAND_POSTCODES`; `:223-228` `SCOTLAND_BLACKLISTED_PRODUCTS`
- `app/Controllers/Http/DeliveryController.js:399` postcode trim; `:461-463` Glasgow lead-time
  rule; `:581-594` today-blocked filter; `:850` postcode rules; `:860` cut-offs
- `app/Services/delivery/deliveryTypes.js:54-77` postcode prefix match; `:171-212` 12:00-22:00
  price modifier
- `app/Services/delivery/dates.js:62-69` commented-out collection check; `:208-245` Specific Day
  blocked dates
- `app/Services/delivery/blockDates.js:16-59` `blockCollectionDates` (dead); `:66-79`
  `isDateBlocked`
- `app/Utils/shipping-services.js:58` email "order before 10pm"
- `app/Utils/expected-delivery-date.js:72-73` cut-off minutes (dev-docker only)
- `app/Services/DeliveryCalendarService.js` union of blocked dates (dev-docker only)

**gatsby-website**
- `src/features/checkout/sidebar/delivery/delivery.js:39-44,74-77` 10pm warning
- `src/layout/header/delivery-banner/delivery-banner.js:9` countdown hour
- `src/templates/category/category.js:431` schema.org cut-off
- `src/features/checkout/service-page/delivery-option/delivery-option.js:37-38` shows `details`
- `src/pages/checkout/delivery/index.js:100,127-128` postcode sent; `:160-161,235-236` depot
  blocked dates

**ff-uk-mobile**
- `src/features/delivery/utils/deliveryEstimate.ts:31-37,51-53,93-94` cut-offs
- `src/lib/ukTime.ts` UK clock
- `src/ui/components/WhyDateSheet.tsx:63-64` "Cut-off time" row
- `src/features/checkout/screens/CheckoutScreen.tsx:752-758` shows `details`; `:810-811` 10pm
  warning
- `src/features/delivery/utils/deliveryQuote.ts:151` depot blocked dates

**admin-website-v2**
- `src/pages/content/delivery/delivery-block-dates/add-date-modal/add-date-modal.jsx:115-158`
  blocked-date form (no region field)
