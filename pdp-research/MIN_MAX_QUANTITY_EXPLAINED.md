# "Min quantity" and "max quantity" on a product — what they are, and should the app respect them

The React Native developer sees two fields coming from the backend when a customer buys a product:
a **minimum quantity** and a **maximum quantity**. He already shows a hint in the app when the
customer types a quantity that does not fit. He asked three things:

1. Are these values real? Should the app respect them, or ignore them?
2. Where do they come from — a value someone typed by hand, or the real stock left in the warehouse?
3. Is the answer in `website-api` and `cdn`?

**Data provenance:** everything below was measured on **2026-09-14**, read-only.
Content data from the **dev** content database (`cdn-test` @
`ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`). Order and basket data from the **dev**
MySQL database (`website_test` @ `54.171.181.199`). Code read at `website-api` **`abc662e`**
(branch `mobile-app`), `cdn-graphql-v2` **`c7cc103`** (`master`), `gatsby-website` **`22fe4a6ca`**
(`master`), `ff-uk-mobile` **`0566c88`** (`main`), `admin-website-v2` **`c4287e6`** (`main`).
No writes of any kind were made. Production was not touched.

Companion docs: [`PRICE_TIER_NOT_SHOWING_INVESTIGATION.md`](./PRICE_TIER_NOT_SHOWING_INVESTIGATION.md)
(§10 corrects one statement in it), [`OBLIGATORY_EXTRAS_PRODUCTS.md`](./OBLIGATORY_EXTRAS_PRODUCTS.md),
[`PRODUCT_PAGE_STRUCTURE.md`](./PRODUCT_PAGE_STRUCTURE.md).

---

## TL;DR

**Both values are real, and the app should respect them. Neither has anything to do with warehouse
stock.** They are two content fields that a First Fence staff member types by hand in the admin
panel. `minQuantity` means "minimum order quantity" (a pack size or a trade minimum).
`orderLimitQuantity` means "maximum quantity that can be ordered at once" (for example, one free
sample box per customer).

There is **no warehouse quantity anywhere in this system**. Not in Mongo, not in MySQL, not in SAP.
The only stock information is a true/false checkbox `inStock` plus a "due back" date. First Fence
decided this on purpose — many products ship direct from suppliers, so there is no per-depot number
to hold.

Two things are worth knowing before any more UI work:

- **They are very rare.** Of 3,735 published and visible products on dev, only **13** have a minimum
  above 1, and only **5** have a maximum at all. The app's current hint "Minimum order quantity: 1"
  is shown on the other ~99.6% of products, where it says nothing useful.
- **The backend does not enforce either field.** `website-api` only caps the quantity using a number
  the client itself put in the request body. If the client leaves it out, any quantity is accepted
  and saved. The hint in the app is the real rule, not a second line of defence.

---

## 1. The short answer

| Question | Answer |
|---|---|
| Is min quantity real? | **Yes.** Field `minQuantity` on the product. |
| Is max quantity real? | **Yes.** Field `orderLimitQuantity` on the product. |
| Should the app respect them? | **Yes for min.** For max, see §7 — the app is currently stricter than the website. |
| Is it warehouse stock? | **No.** No stock number exists anywhere (§4). |
| Who sets the value? | A staff member types it in the admin panel (§2). |
| Is it synced from SAP or anywhere? | **No.** Nothing writes these fields except a human (§4). |
| Which repos? | `cdn-graphql-v2` (stores and serves it) and `admin-website-v2` (sets it). `website-api` only passes it through. |
| Where does the app receive it? | **Both** places — the PDP GraphQL query **and** every `GET /carts` line (§6). |

The developer's guess about the repos was close but not complete. The value is **stored** in the
`cdn` Mongo database and **served** by `cdn-graphql-v2`. It is **set** in `admin-website-v2`.
`website-api` never looks it up — it only reads it back out of the request body, and it re-serves it
on the cart because of a generic object spread.

---

## 2. Where the values come from

Both fields are plain number inputs on the product edit page in the admin panel.

**Admin URL:** `/content/products/:id` → *General* tab.

`admin-website-v2/src/pages/content/product/main-fields/main-fields.jsx:173-187`:

```jsx
<NumberField value={minQuantity} field={'minQuantity'}
  label={'Min Quantity'} tooltip={TOOLTIP.MIN_QTY} />

<NumberField value={orderLimitQuantity} field={'orderLimitQuantity'}
  label={'Order Limit Quantity'} tooltip={TOOLTIP.ORDER_LIMIT_QTY} />
```

The tooltips staff see, quoted exactly from
`admin-website-v2/src/utils/tooltip-messages.js:13-18`:

| Field | Label in admin | Tooltip text |
|---|---|---|
| `minQuantity` | Min Quantity | *"Products should have quantity 1 except the products in group where you could have products with value of 0!"* |
| `orderLimitQuantity` | Order Limit Quantity | *"Maximum quantity that can be ordered at once."* |

The second tooltip is the clearest statement of intent we have. It says **"ordered at once"**, not
"available". It is an order rule, not an availability number.

There is a third, separate field at the option level.

**Admin URL:** `/content/product-variants/:id` → options table → edit option modal.
`admin-website-v2/src/pages/content/product-variants/product-variant-options/edit-option-modal/edit-option-modal.jsx:114-134`:

```jsx
<OptionalField label={'Minimum Order Quantity'} field={'minQuantity'}
  tooltip={'Minimum order quantity required when this option is selected'} ... >
```

### What the admin screen does NOT do

- **No validation at all.** The shared `NumberField`
  (`admin-website-v2/src/atoms/number-field/number-field.jsx`) has no `min`, no `max`, no `required`
  and no validator. It only forbids negative numbers and decimals.
- **No cross-check.** Nothing stops a staff member setting a maximum lower than the minimum, or a
  maximum lower than the cheapest price tier. That second mistake has already happened once — see §10.
- **No import, no sync.** The bulk product uploader does not touch these fields (searched
  `quantity` and `qty` across `admin-website-v2/src/pages/content/bulk-product-uploader/` — zero hits).

---

## 3. What the data actually looks like on dev

Measured on `cdn-test`, 2026-09-14.

| Measure | Count |
|---|---|
| Products in the collection | 7,905 |
| Status `PUBLISHED` | 5,574 |
| `PUBLISHED` **and** `visible: true` (what a shopper can reach) | **3,735** |
| `minQuantity` field present | 7,905 (all of them, never null) |
| `minQuantity` greater than 1 — all statuses | 25 |
| `minQuantity` greater than 1 — published **and** visible | **13** |
| `orderLimitQuantity` set to a number | **5** |
| `orderLimitQuantity` explicitly null | 3,525 |
| `orderLimitQuantity` key not present at all | 4,375 |

So the minimum is a real rule on **0.35%** of the shop, and the maximum on **0.13%**.

### The 13 products with a real minimum

| SKU | Title | min | Price |
|---|---|---|---|
| `TRAF-MAT-0161` | EnduraGrid™ Carpark Line Marker - Yellow | 240 | £2.38 |
| `TRAF-BIN-0075` | 450kg Yellow Grit Bin/Storage Bin | 14 | £274.75 |
| `TRAF-MIS-0005` | Wychwood Hazard Verge Post - Cables Overhead | 10 | £22.94 |
| `TRAF-MIS-0003` | Wychwood Hazard Verge Post - Cables Underground | 10 | £22.94 |
| `MISC-BOL-0180` … `MISC-BOL-0185` | 900mm / 1000mm stainless steel bollards (6 SKUs) | 10 | £93 – £228 |
| `MISC-BOL-0187`, `MISC-BOL-0188` | Sheffield Cycle Stand (dig in / bolt down) | 10 | £87 – £114 |
| `TRAF-TRE-0105` | Oxford Low-Pro Cover 1200mm x 800mm | 6 | £133.56 |

These read like trade minimums and pack sizes. A £2.38 line marker sold in 240s is a box. Bollards
sold in 10s is a normal trade minimum.

### All 5 products with a maximum

| SKU | Title | max | Price |
|---|---|---|---|
| `COMPOSITE-FENC-9500` | Free Bambura® Composite **Fencing Samples Box** | 1 | £0.00 |
| `COMPOSITE-SAMPLE-1000` | Free Bambura® Composite **Decking Samples Box** | 1 | £0.00 |
| `GRE-ACCS-GATE-5395` | Gatemaster Quick Exit Keylatch Lock Kit | 1 | £143.10 |
| `FF-WEB-0438` | PaliGuard® HD Fencing Pale to Suit 3.0m | 2 | £15.79 |
| `ROLL-CEN-00038` | 48mm Plain Nylon Roller With Bearing | 20 | £11.44 |

**Two of the five are free sample boxes priced at £0.00 with a limit of 1.** That is the plainest
possible proof of meaning: it is "one free sample per customer", a business rule. Nobody has 1 roller
bearing left in a warehouse and 1 free sample box left at the same time.

### The option-level minimum — exactly one product uses it

Across 4,564 variants and **31,994** options, only **9** options carry a minimum. All 9 are in a
single variant document (`602bcadff77e7453275ad68a`), used by a single product:
**Galfan 358 Mesh Sheets - Panels Only** (`WEB-01393`).

The `stockMessage` text on those options was typed by a human and says it outright:

| Option | `minQuantity` | `stockMessage` |
|---|---|---|
| 1270mm | 50 | *"Minimum Order Quantity of 50"* |
| 1829mm – 2997mm | 25 | *"Minimum Order Quantity of 25"* |
| 3302mm | 20 | *"Minimum Order Quantity of 20"* |
| 4205mm | 15 | *"Minimum Order Quantity of 15"* |
| 5207mm | 10 | *"Minimum Order Quantity of 10"* |

Note the pattern: the **shorter** the sheet, the **higher** the minimum. That is a packing and
haulage rule, and it is the opposite of what stock would look like.

> Careful with the field name. `stockMessage` is not about stock. It is a free-text helper line on an
> option. The most common values on dev are things like *"Use to start or terminate a run."* and
> *"Add for each corner needed for the fence."*

---

## 4. It is not warehouse stock — the proof

This is the part of the question that matters most, so here is the evidence from every direction.

**1. There is no numeric stock field in the content database.** I listed every key used by the
`products` collection on dev. The only stock-related keys are `inStock` (boolean),
`inStockGuaranteed` (boolean), `notInStock` (boolean, deprecated) and `dateDueBackInStock` (a date).
There is no count.

**2. There is no stock table in MySQL.** Every column in `website_test` whose name contains
`stock`, `quantit`, `qty`, `inventor` or `limit`:

| Table | Column | Type |
|---|---|---|
| `products` | `inStock` | `tinyint(1)` — a boolean copied onto the basket line |
| `products` | `qty` | `int` — the quantity the customer ordered |
| `product_options` | `qty` | `int` |
| `extras` | `qty` | `int` |
| `delivery_types` | `weight_limit` | `float(8,2)` — delivery weight, unrelated |
| `installation_surcharges` | `modifier_quantity` | `float(8,2)` |
| `temporary_works_designs` | `quantity` | `int unsigned` |

No stock table, no inventory table, no `orderLimitQuantity` column.

**3. SAP is never asked for stock.** `website-api/app/Services/SAPService.js` runs about 30 SQL
queries against SAP HANA. The tables it touches are `ORDR, OCRD, OINV, RDR1, RDR12, OQUT, QUT1,
QUT12, ODPI, OSHP, OHEM, ADOC, OWHS, OSLP, OCPR, OCRY, OPLN, CRD1` — orders, invoices, customers,
sales reps, depots, credit. The SAP Business One stock tables are `OITW` and `OITM`, with columns
`OnHand`, `IsCommited`, `OnOrder`. **None of them appear anywhere in the repo.**

**4. Salesforce does not sync stock.** Every Salesforce call in `website-api` is an outbound write of
a Task or a Case.

**5. The business already decided this.** `website-api/.generated_docs/FirstFence-Meetings.md:579-582`:

> *"A meaningful chunk of the catalogue isn't kept as physical stock at First Fence depots — many
> products come direct from suppliers. The Discovery document explicitly notes this: 'Live depot
> stock per depot — Not feasible — many products come direct from suppliers'. So even if the system
> tried to reserve stock, there's no reliable per-depot stock figure to reserve against."*

The same file (`:565-566`) confirms a quote never reserves stock.

**Conclusion: min and max quantity are commercial rules. Stock is a separate, much cruder concept —
a yes/no flag with an optional "back on" date.**

---

## 5. What each client does with the two fields today

This is where the three apps disagree with each other.

| Behaviour | Mobile app | Website | `website-api` |
|---|---|---|---|
| Reads `minQuantity` | Yes | Yes | Only on reorder |
| Below minimum on the PDP | **Silently rejected**, value snaps back, no message | **Silently clamped** up on blur, no message | Not checked |
| Below minimum in the basket | **Ignored**, floor is a hardcoded 1 | Blocked by the stepper, snaps up on blur | Not checked |
| Reads `orderLimitQuantity` | Yes | Yes, but only to forward it | Only from the request body |
| Above maximum | **Silently clamped** down, no message | **Nothing at all** | Clamps only if the client sent the number |
| Option-level minimum | Message **and** button disabled | Message **and** button disabled | Not checked |
| Any warehouse stock check | No | No | No |

### Mobile

`ff-uk-mobile/src/core/pricing/configurator.ts:554-559`:

```ts
case 'setQuantity': {
  let quantity = Math.floor(action.value);
  if (Number.isNaN(quantity) || quantity < product.minQuantity) return state;
  if (product.orderLimitQuantity && quantity > product.orderLimitQuantity) {
    quantity = product.orderLimitQuantity;
  }
```

The hint the developer mentioned is at
`ff-uk-mobile/src/features/product/blocks/QuantityBlock.tsx:148`:

```tsx
helper={`Minimum order quantity: ${product.minQuantity}`}
```

**It has no condition on it.** `mapProductDetail.ts:323` defaults the value to 1
(`Math.max(1, raw.minQuantity ?? 1)`), so the app prints *"Minimum order quantity: 1"* on roughly
3,722 of 3,735 products. See §7 for what to do about that.

The only real error message comes from the **option** level, at
`ff-uk-mobile/src/core/pricing/configurator.ts:343`:

```ts
message: `Minimum order quantity is ${optionMinQty} for the selected option.`
```

That one also disables the Add to Basket button
(`ff-uk-mobile/src/features/product/components/AddToBasketBar.tsx:178,203`). As shown in §3, exactly
one product on dev can trigger it.

The basket ignores both fields completely —
`ff-uk-mobile/src/store/slices/basketSlice.ts:151` is `Math.max(1, action.payload.quantity)`. A
product with a minimum of 10 can be reduced to 1 in the basket.

### Website

The website enforces the minimum (as a clamp) but **never enforces the maximum**. I searched the
whole repo: `orderLimitQuantity` appears 9 times, and every one is either a GraphQL field name, the
model parsing it, or the line that puts it in the cart request
(`gatsby-website/src/models/Product.js:627`). There is not one comparison against it. The quantity
input cannot even express a maximum — `gatsby-website/src/components/number-field/number-field.js:48`
passes `min` and has no `max` prop.

### `website-api`

`website-api/app/Controllers/Http/CartController.js:298-303` reads the cap **out of the request**:

```js
const { options, installationType, orderLimitQuantity, mongo_id: mongoId, qty } = product;
```

and then `:333` does `const finalQty = Math.min(newQty, orderLimitQuantity);`, guarded by
`if (orderLimitQuantity && orderLimitQuantity > 0 && mongoId)`. The same pattern is in
`saveProduct` at `:405-409`, and in `ProductController.js:425-433`.

**The API never looks the real value up.** If the client leaves the field out of the POST body, no
cap is applied and any quantity is saved. `minQuantity` is never checked on any write path at all —
it appears exactly once in the whole backend, at `app/Utils/reorder-changes.js:210`.

The one place the backend reads the real values from Mongo is **reorder** and **quote validate**
(`website-api/app/Utils/reorder-changes.js:204-213`):

```js
let quantity = line.qty;
const cap = doc.orderLimitQuantity;
if (cap && cap > 0 && quantity > cap) { quantity = cap; facets.qtyCapped = { to: cap }; }
const min = doc.minQuantity;
if (min && min > 1 && quantity < min) { quantity = min; facets.qtyRaised = { to: min }; }
```

This only changes the payload that is returned. It never writes
(`app/Services/ReorderValidationService.js:16`). Note `min > 1`, so a minimum of 1 is a deliberate
no-op, and the cap is applied before the minimum, so if someone sets a minimum higher than the
maximum the minimum wins.

---

## 6. The exact answer to "is it the basket or the PDP?"

The developer was unsure which. The answer is **both carry the fields, but only the PDP acts on them.**

- **PDP:** requested directly from the CDN GraphQL API —
  `ff-uk-mobile/src/features/product/services/productDetail.graphql:23-24`.
- **Basket:** `GET /carts` returns them too, by accident. `CartController.js:91` builds each line as:

  ```js
  products.push({ ...found, ...product, productVariants, installationType });
  ```

  `found` is the live Mongo product document, spread first. So every Mongo-only field — including
  `minQuantity` and `orderLimitQuantity` — passes straight through into the cart response, even
  though the MySQL cart row has no such columns and the API does not use them. The MySQL `qty` wins
  because `product` is spread second.

So if the developer sees the fields on a basket response, that is real, but it is disclosure, not a
rule the backend is applying.

---

## 7. Traps to know before changing anything

| # | Trap | Detail |
|---|---|---|
| 1 | **`0` means "no limit", not "cannot order"** | `admin-website-v2/src/atoms/number-field/number-field.jsx:26-32` writes `0` when a staff member clears the box. `gatsby-website/src/models/Product.js:42-44` and `configurator.ts:557` both treat `0` as falsy, so a cleared field disables the limit. That is the safe direction, but it means "set the limit to 0 to block sales" silently does nothing. |
| 2 | **The app is stricter than the website on max** | The app clamps to `orderLimitQuantity`; the website ignores it. Same product, two behaviours. Before adding a visible max message in the app, confirm with the business that the limit is meant to be enforced at all — otherwise QA will report the app as broken against the website. |
| 3 | **The backend is not a safety net** | The cap only fires if the client sends `orderLimitQuantity` in the POST body. Do not rely on the API to protect the rule. |
| 4 | **The hint fires on every product** | The `Minimum order quantity: 1` helper is unconditional. Suggest showing it only when `product.minQuantity > 1`. That is a one-line change in `QuantityBlock.tsx:148`, and it belongs to the mobile repo, not to me. |
| 5 | **`minQuantity` has a second, unrelated meaning** | On an extra/option, `minQuantity > 0` is currently the only signal the business has for "this extra is mandatory". See `website-api/.generated_docs/epic-3-checkout/FIR-16-Cart-Session-Refactor-Strategy.md:204-209` and [`OBLIGATORY_EXTRAS_PRODUCTS.md`](./OBLIGATORY_EXTRAS_PRODUCTS.md) §5-6. Do not repurpose the field. |
| 6 | **Website and app can disagree for weeks** | `gatsby-website/.gitlab-ci.yml:37` says *"Gatsby build must be triggered manually from the admin site"*. A min/max change in admin reaches the **app immediately** (live GraphQL) but reaches the **website only after a manual rebuild**. If QA compares the two, check the build date first. |
| 7 | **The minimum can be raised above what customers already bought** | See §8. Raising a minimum does not touch existing baskets, so old lines stay below it. |

---

## 8. When did these values appear? (the timeline)

The dev content database has an audit collection, `productlogs`, with 5,725 entries. It stops dead on
**2025-12-17 15:38:05**, because that is the day the logging was switched off:

```
d66790a  2025-12-17  Martin Holecek  Remove history logs to save space and time
```

All three `ProductLog` writes in `cdn-graphql-v2/src/resolvers/product.js` (lines 100, 565, 617) are
commented out by that commit.

This produces a clean, and slightly unlucky, timeline:

| Date | Event |
|---|---|
| 2025-12-17 | Product change logging turned off (`d66790a`, `cdn-graphql-v2`) |
| **2026-01-22** | The "Order Limit Quantity" field is **added to the admin form** (`6df6225`, `admin-website-v2`), and the same day the website starts carrying it (`f46b74f7d`, `gatsby-website`) |
| 2026-05-29 | Option-level minimum built end to end across admin and website |
| 2026-07-23 | The mobile clamp and the "Minimum order quantity: N" helper land (`87e7159`, `4facf52`, `ff-uk-mobile`) |

**So the maximum only became settable one month after the audit log was turned off.** That is why no
log entry in the collection carries an `orderLimitQuantity` key at all — I checked, the count is 0,
while `minQuantity` is present in 5,713 of 5,725 snapshots. We cannot date any of the five limits
from the log. This is a real gap, not a mystery.

The minimum *can* be dated, for the two products edited inside the logging window, and it proves the
value is typed by hand:

| When | Who | SKU | Change |
|---|---|---|---|
| 2025-11-26 15:10 | user `28` | `TRAF-TRE-0105` | `minQuantity` **1 → 6** |
| 2025-12-17 07:43 | user `28` | `TRAF-BIN-0075` | `minQuantity` **7 → 14** |

Both are `operationCode: UPDATE_PRODUCT`. `TRAF-BIN-0075` was touched four times in 29 minutes by two
different users — a person adjusting a number, not a job writing one.

> One caution. Every log row has `sapSync: false`, but that field is **hardcoded** `false` in the
> (now commented-out) write. It is not evidence about SAP. The evidence about SAP is §4 point 3.

### Did raising a minimum break anything?

This was the obvious alternative explanation to rule out: maybe the minimum is not enforced at all,
because plenty of basket lines sit below it. Checked against 1,794,296 basket lines in `website_test`:

| SKU | min | Lines below the minimum | After the minimum was raised |
|---|---|---|---|
| `TRAF-TRE-0105` | 6 | 95 of 109 | Only 2 lines exist after the edit at 15:10, and **both are exactly 6** |
| `TRAF-BIN-0075` | 14 | 19 of 19 | **Zero lines exist after 2025-12-17** |
| `MISC-BOL-0180` … `0188` (8 SKUs) | 10 | **0 of 227** | smallest quantity ever seen is exactly 10 |

(`TRAF-TRE-0105` has a third line dated 2025-11-26 10:39 with quantity 1. That is the same day, but
**before** the 15:10 edit, so it was legal when it was created.)

So the minimum **is** respected. The apparent violations are historical lines created when the
minimum was still 1 or 7. The bollards, whose minimum was never changed, have never once been bought
below 10 in 227 basket lines. That is the control group.

### Is the maximum respected? No — 9 lines break it

| Line id | SKU | max | qty ordered | Created | Cart completed? |
|---|---|---|---|---|---|
| 267818 | `ROLL-CEN-00038` | 20 | 25 | 2023-04-03 | yes |
| 2198202 | `COMPOSITE-FENC-9500` | 1 | 5 | 2026-01-20 | yes |
| 2198203, 2198208 | `COMPOSITE-FENC-9500` | 1 | 5 | 2026-01-20 | no |
| 2314080 | `FF-WEB-0438` | 2 | 16 | 2026-03-10 | no |
| 2352819 | `FF-WEB-0438` | 2 | 8 | 2026-04-30 | no |
| 2358028 | `FF-WEB-0438` | 2 | 5 | 2026-05-07 | no |
| 2390440 | `ROLL-CEN-00038` | 20 | 150 | 2026-06-11 | no |
| 2393599 | `ROLL-CEN-00038` | 20 | 150 | 2026-06-15 | no |

Three of these were **paid for** (`completed = 1`), including 5 free sample boxes on one order where
the limit is 1.

But I cannot claim all 9 were breaches at the time. `ROLL-CEN-00038` was last updated **2026-06-19**,
four days *after* the two 150-quantity lines, and it has no price tiers — so the limit of 20 was most
likely typed **in response to** that test, not before it. Because logging is off (see above), this
cannot be proven either way. What the table does prove is the mechanism: nothing in the website and
nothing in the API reliably stops a quantity above the limit.

---

## 9. Is this just dev? — three separate questions

**a. Can a deploy or pipeline create these values on production?**
No. These are content values typed by a person in the admin panel, against whichever Mongo database
that admin instance points at. No migration, seeder, importer or CI job writes them — I searched the
bulk uploader and the migrations. The only code-level default is in
`cdn-graphql-v2/src/resolvers/product.js:61-64`, which sets `minQuantity` to 1 and
`orderLimitQuantity` to `null` on create. That default is safe.

**b. Can it happen again after the fixes already in the repo?**
Yes. There are no fixes in the repo for this. Any staff member with admin access can type any number
into either box at any time, with no validation, no cross-check against price tiers, and — since
2025-12-17 — **no audit trail**. The `FF-WEB-0438` limit of 2 that broke a price tier is still set
today (the product was last updated 2026-09-03), even though a previous investigation recommended
clearing it.

**c. Is production already in that state?**
**I cannot tell you, and I did not look.** Production was not touched. There are no production
credentials in `website-api/.env`, and the rule for this work is not to read production at all.

One earlier doc states that the production copy of `FF-WEB-0438` has `orderLimitQuantity = null`
(`PRICE_TIER_NOT_SHOWING_INVESTIGATION.md:300`), but I did not re-verify that, and it only covers one
product.

To settle it yourself, run this read-only query against the production Mongo, filling in the
production host, database and credentials:

```bash
docker exec -i firstfence-mongo mongosh --quiet \
  "mongodb://<user>:<pass>@<PROD_HOST>:27017/<PROD_DB>?authSource=<PROD_DB>" --eval '
const c = db.getCollection("products");
print("published+visible: " + c.countDocuments({status:"PUBLISHED", visible:true}));
print("min > 1:           " + c.countDocuments({status:"PUBLISHED", visible:true, minQuantity:{$gt:1}}));
print("order limit set:   " + c.countDocuments({status:"PUBLISHED", visible:true, orderLimitQuantity:{$ne:null,$exists:true}}));
c.find({status:"PUBLISHED", visible:true, orderLimitQuantity:{$ne:null,$exists:true}},
       {sku:1,title:1,minQuantity:1,orderLimitQuantity:1,price:1,updated:1}).forEach(p=>printjson(p));
'
```

If production returns a similar handful of products, the behaviour described here is the real
behaviour for real customers, not a dev artefact.

---

## 10. How this fits with the docs we already have

| Doc | Status |
|---|---|
| [`PRODUCT_PAGE_STRUCTURE.md`](./PRODUCT_PAGE_STRUCTURE.md) `:572`, `:937` | **Still correct.** It already says *"There is no upper limit on the steppers — `orderLimitQuantity` exists in the data and is sent to the cart but is never enforced or shown."* My measurement agrees for the website. |
| [`OBLIGATORY_EXTRAS_PRODUCTS.md`](./OBLIGATORY_EXTRAS_PRODUCTS.md) §5 | **Still correct.** It names exactly one product using option-level minimums (`WEB-01393`, Galfan 358 Mesh Sheets). I counted the same: 9 options, 1 variant, 1 product. |
| `website-api/.generated_docs/epic-2-products/FIR-58-PDP-API-Backend-Assessment.md:29,35` | **Still correct.** It lists both fields as *"Fully present. No backend work"* — read-side metadata for the client, not a server rule. |
| `website-api/.generated_docs/epic-3-checkout/FIR-21-FIR-157-Reorder-Backend-Strategy.md:347-357` | **Still correct**, and the most precise prior statement: *"`orderLimitQuantity` is not a column on `products` … It is read from the request payload purely to cap qty at write time."* |
| [`PRICE_TIER_NOT_SHOWING_INVESTIGATION.md`](./PRICE_TIER_NOT_SHOWING_INVESTIGATION.md) | **Needs two corrections.** See below. |

### Corrections to `PRICE_TIER_NOT_SHOWING_INVESTIGATION.md`

**1. Line 290-292 overstates the backend guarantee.** It says:

> *"Basket API — `website-api/app/Controllers/Http/CartController.js` and `ProductController.js` both
> do `Math.min(qty, orderLimitQuantity)`. So even a hand-made request cannot get 2000 into the
> basket."*

The second sentence is wrong. Both controllers take `orderLimitQuantity` **from the request body**
(`CartController.js:298-303`), not from Mongo. A hand-made request that simply omits the field skips
the `if (orderLimitQuantity && orderLimitQuantity > 0)` guard, and any quantity is saved. §8 shows
basket lines of 150 against a limit of 20 and 16 against a limit of 2.

**2. Line 303-306 is right but understates the gap.** It says logging is switched off and the
`productlogs` collection has *"0 entries for this product"*. Both true. The fuller picture: the
collection holds **5,725 entries covering 2025-11-21 to 2025-12-17** and then stops, at commit
`d66790a`. And because the order limit field was only added to the admin on 2026-01-22, **no log
entry anywhere carries an `orderLimitQuantity` value** — so no order limit on dev can ever be dated
from the log.

**One thing has changed since that doc was written (2026-09-03).** Its price tiers on `FF-WEB-0438`
are no longer the ones described. The cheapest tier is now **2500 → £9.00**, not 2000 → £10. The
`orderLimitQuantity = 2` is still there, so the contradiction that doc identified is **not fixed**.

---

## 11. What I did not verify

Stated plainly, so nothing here is mistaken for measurement.

- **Production, in any form.** No production database, no production API, no production admin. Every
  number in this doc is from dev. §9c has the command to settle the production question.
- **A live `GET /carts` response.** I read the code path that merges the Mongo document into the cart
  line (`CartController.js:91`) but did not call the dev API with a real session to see the JSON.
- **Who user `28`, `52` and `137` are.** The audit log stores a numeric id only. I did not map them to
  people.
- **When the five order limits were set.** Not knowable from the data — see §8. The 2026-06-19
  timing for `ROLL-CEN-00038` is an inference from the `updated` date, not a measurement.
- **Whether the business *wants* the maximum enforced.** That is a product decision, not a technical
  one. The tooltip says it is a maximum; the website does not treat it as one. Somebody at First Fence
  should confirm which is intended.
- **SAP HANA directly.** I did not connect to it. The conclusion that SAP holds no stock for this
  system is based on reading every query in `SAPService.js` and searching for the SAP stock tables
  (`OITW`, `OITM`, `OnHand`, `IsCommited`, `OnOrder`) across the repo. SAP may well hold stock data —
  the point is that **nothing in our code ever asks for it**.
- **The `notInStock` field.** Marked deprecated in the schema
  (`cdn-graphql-v2/src/schema/product.js:202`). I did not trace whether anything still reads it.

---

## Appendix A — queries used

**Dev Mongo** (`cdn-test` @ `ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`), run through the
local container:

```bash
docker exec -i firstfence-mongo mongosh --quiet \
  "mongodb://firstfence:<pass>@ec2-34-252-129-247.eu-west-1.compute.amazonaws.com:27017/cdn-test?authSource=cdn-test" \
  --eval "$(cat script.js)"
```

Field coverage:

```js
const c = db.getSiblingDB('cdn-test').products;
c.countDocuments({});                                              // 7905
c.countDocuments({status:"PUBLISHED", visible:true});              // 3735
c.countDocuments({minQuantity:{$gt:1}});                           // 25
c.countDocuments({status:"PUBLISHED", visible:true, minQuantity:{$gt:1}});   // 13
c.countDocuments({orderLimitQuantity:{$ne:null,$exists:true}});    // 5
c.countDocuments({orderLimitQuantity:{$exists:false}});            // 4375
```

Value distribution and detail:

```js
c.aggregate([{$group:{_id:"$minQuantity", n:{$sum:1}}},{$sort:{_id:1}}]);
c.find({status:"PUBLISHED", visible:true, minQuantity:{$gt:1}},
       {title:1,sku:1,minQuantity:1,orderLimitQuantity:1,price:1,unit:1,inStock:1});
c.find({orderLimitQuantity:{$ne:null,$exists:true}},
       {title:1,sku:1,minQuantity:1,orderLimitQuantity:1,price:1,unit:1,created:1,updated:1});
```

Option-level minimums:

```js
const v = db.getSiblingDB('cdn-test').productvariants;
v.aggregate([{$unwind:"$options"},
  {$group:{_id:null, totalOptions:{$sum:1},
           withMin:{$sum:{$cond:[{$gt:["$options.minQuantity",0]},1,0]}}}}]);
// { totalOptions: 31994, withMin: 9 }
v.aggregate([{$unwind:"$options"},{$match:{"options.minQuantity":{$gt:0}}},
  {$project:{optName:"$options.name", min:"$options.minQuantity",
             msg:"$options.stockMessage"}}]);
db.getSiblingDB('cdn-test').products.find({productVariants:"602bcadff77e7453275ad68a"},
  {title:1,sku:1,status:1,visible:1});   // -> WEB-01393 only
```

Audit log:

```js
const L = db.getSiblingDB('cdn-test').productlogs;
L.countDocuments({});                                        // 5725
L.find({}).sort({_id:1}).limit(1);                           // first: 2025-11-21
L.find({}).sort({_id:-1}).limit(1);                          // last:  2025-12-17
L.countDocuments({"previous.orderLimitQuantity":{$exists:true}});  // 0
L.countDocuments({"previous.minQuantity":{$exists:true}});         // 5713
L.find({$expr:{$ne:["$current.minQuantity","$previous.minQuantity"]}},
       {user:1, operationCode:1, "previous.sku":1,
        "previous.minQuantity":1, "current.minQuantity":1});
```

**Dev MySQL** (`website_test` @ `54.171.181.199`), read-only:

```bash
docker exec -i firstfence-mysql mysql -h 54.171.181.199 -u mobile_dev -p'<pass>' website_test -e "<sql>"
```

Schema sweep for any stock or limit column:

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='website_test'
  AND (COLUMN_NAME LIKE '%stock%' OR COLUMN_NAME LIKE '%quantit%'
       OR COLUMN_NAME LIKE '%qty%' OR COLUMN_NAME LIKE '%inventor%'
       OR COLUMN_NAME LIKE '%limit%')
ORDER BY TABLE_NAME, COLUMN_NAME;
```

Basket lines that break an order limit:

```sql
CREATE TEMPORARY TABLE lim (mongo_id VARCHAR(200) PRIMARY KEY, sku VARCHAR(64), lim INT);
INSERT INTO lim VALUES
 ('612e39ef59ca05271c2d0e98','ROLL-CEN-00038',20),
 ('68c2e141ce86564f7092c996','COMPOSITE-FENC-9500',1),
 ('698f18bf2d4a743dbae14c8e','FF-WEB-0438',2),
 ('69fdbd448292e69fb62b93b2','COMPOSITE-SAMPLE-1000',1),
 ('6a315c82ab2829f13a1d2dda','GRE-ACCS-GATE-5395',1);

SELECT p.id, l.sku, l.lim AS orderLimit, p.qty, p.priceEa,
       ROUND(p.qty*p.priceEa,2) AS line_value, p.created_at, c.completed
FROM lim l
JOIN products p ON p.mongo_id = l.mongo_id
LEFT JOIN carts c ON c.id = p.cart_ID
WHERE p.qty > l.lim
ORDER BY p.created_at;
```

Basket lines below a minimum, split before and after the minimum was raised:

```sql
-- TRAF-TRE-0105, minQuantity raised 1 -> 6 on 2025-11-26
SELECT CASE WHEN p.created_at < '2025-11-26' THEN 'BEFORE' ELSE 'AFTER' END AS period,
       COUNT(*) AS n_lines, SUM(p.qty<6) AS below_min, MIN(p.qty) AS min_qty
FROM products p WHERE p.mongo_id='5f16f3ba8b49590d02de95f6' GROUP BY period;

-- TRAF-BIN-0075, minQuantity raised 7 -> 14 on 2025-12-17
SELECT CASE WHEN p.created_at < '2025-12-17' THEN 'BEFORE' ELSE 'AFTER' END AS period,
       COUNT(*) AS n_lines, SUM(p.qty<14) AS below_min, MIN(p.qty) AS min_qty
FROM products p WHERE p.mongo_id='60094e91db710e30fa91080c' GROUP BY period;
```

## Appendix B — the files that matter

| Repo | File:line | What it does |
|---|---|---|
| `cdn-graphql-v2` | `src/models/product.js:15-16` | Declares `minQuantity`, `orderLimitQuantity` |
| `cdn-graphql-v2` | `src/models/product-variants.js:24` | Option-level `minQuantity` |
| `cdn-graphql-v2` | `src/resolvers/product.js:61-64` | Defaults on create: min → 1, limit → `null` |
| `cdn-graphql-v2` | `src/resolvers/product.js:100,565,617` | Audit log writes, commented out since 2025-12-17 |
| `admin-website-v2` | `src/pages/content/product/main-fields/main-fields.jsx:173-187` | The two admin inputs |
| `admin-website-v2` | `src/utils/tooltip-messages.js:13-18` | The tooltips staff read |
| `admin-website-v2` | `src/pages/content/product/normalize.js:155-156` | Saved as `getInt(…, 1)` and `getInt(…, null)` |
| `admin-website-v2` | `src/atoms/number-field/number-field.jsx:26-32` | Cleared box writes `0` |
| `ff-uk-mobile` | `src/core/pricing/configurator.ts:554-559` | The only min/max rule in the app |
| `ff-uk-mobile` | `src/core/pricing/configurator.ts:336-345` | Option-level min error |
| `ff-uk-mobile` | `src/features/product/blocks/QuantityBlock.tsx:148` | The unconditional hint |
| `ff-uk-mobile` | `src/store/slices/basketSlice.ts:151` | Basket floor of 1, ignores both fields |
| `gatsby-website` | `src/models/Product.js:42-44,385,627` | Parses, clamps min, forwards limit |
| `gatsby-website` | `src/utils/product/validation.js:30-36` | Option-level min blocks Add to Basket |
| `website-api` | `app/Controllers/Http/CartController.js:298-303,333,405-409` | Cap taken from the request body |
| `website-api` | `app/Controllers/Http/CartController.js:91` | Mongo spread — why the fields appear on cart lines |
| `website-api` | `app/Utils/reorder-changes.js:204-213` | The only rule read from the real product |
