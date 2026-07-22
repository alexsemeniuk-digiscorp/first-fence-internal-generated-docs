# Zero-price products: "Free" vs "Price on Application" — verified findings

An addendum to `PRODUCT_PAGE_STRUCTURE.md` and `PRODUCT_PAGE_STRUCTURE_SCOPED.md`.
Neither of those files has been changed. This one supersedes their short "No price / Free" notes.

**Why this exists.** The structure docs say a product whose `price` is `0` shows the word **"Free"** in the
price block. The designer could not find such a product on the live site, and the one zero-price product
they did find (`pyramid-barrier-1-metre`, dev only) showed **"PRICE ON APPLICATION"** instead. That looked
like the doc was wrong.

**Verdict: the doc is correct, but it was unusable.** It never named a live example, and it never explained
that `price = 0` has *three completely different outcomes* depending on two other fields. It also never
warned that "Price on Application" has nothing to do with `price` at all. Everything below was verified
against the production Mongo dump (7,885 products) **and** against live production HTML.

---

## 1. The short answer

`price = 0` on its own tells you nothing. Three fields decide what a shopper sees:

| What the shopper sees | Requires |
|---|---|
| **"Free"** | `price` is `0` **and** `ignoreBasePrice` is off |
| **No price block at all** | `ignoreBasePrice` is on (whatever `price` is) |
| **"PRICE ON APPLICATION"** | `productType` is `POA` — **independent of `price`** |

"Free" is real, live, and rare. "Price on Application" is a different page template entirely and is never
triggered by a zero price.

---

## 2. "Free" — confirmed live, three products

These are the **only** three products on production that render "Free" today. All verified by fetching the
production page:

| Path | SKU | Title |
|---|---|---|
| `/envirorail-bow-top-sample` | `WEB-08303` | Mini EnviroRail Bow Top Railing Sample |
| `/free-bambura-wood-plastic-composite-fencing-samples-box` | `COMPOSITE-FENC-9500` | Free Bambura® WPC Fencing Samples Box |
| `/free-bambura-wood-plastic-composite-decking-samples-box` | `COMPOSITE-SAMPLE-1000` | Free Bambura® WPC Decking Samples Box |

They are all **free sample boxes** — which is the entire business reason the state exists. There is no
free *product*; there are free *samples*.

### What the page actually renders

Extracted from live production HTML for `/envirorail-bow-top-sample`:

```
Mini EnviroRail Bow Top Railing Sample     ← h1
Free  EACH                                 ← price block: the word, then the unit
☑ In Stock                                 ← stock line, unchanged
Pay in 3 Payments at 2.5% interest with…   ← finance line, unchanged
```

Points that matter for the mobile design:

- The word **"Free"** occupies the exact slot the big price figure normally occupies, at the same size and
  weight. It is not a badge, not a pill, not coloured differently. Just the word where "£158.39" would be.
- The **unit word still renders** next to it — "Free EACH". Reads awkwardly; worth designing around
  rather than reproducing.
- The **"£X incl VAT" line is suppressed.** The block is one line shorter than the normal state.
- Nothing else on the page changes. Stock line, finance line, gallery, Add To Basket all behave normally.

### The inconsistency you will hit

"Free" appears **only** in the header price block. Everywhere else on the same page the number is
formatted as an ordinary price:

| Where | Live text on the free-sample page |
|---|---|
| Header price block | `Free` `EACH` |
| Add To Basket box | `£0.00` `EA` … `£0.00 incl VAT` |
| Sticky Add To Basket bar | `£0.00` `TOTAL` … `£0.00 incl VAT` |

So the same product simultaneously says "Free" at the top and "£0.00 incl VAT" further down. **Decide once
for the mobile app** whether zero is "Free" everywhere or "£0.00" everywhere — the web is inconsistent and
copying it will look like a bug.

---

## 3. `ignoreBasePrice` — the far more common zero-price case

This is what the designer was most likely hitting when browsing zero-price products, and the structure docs
buried it in a single bullet.

Of the 42 published, visible, zero-price products that get a page built:

- **39** have `ignoreBasePrice: true` → **no price block at all**
- **3** have `ignoreBasePrice: false` → **"Free"** (the ones in section 2)

When `ignoreBasePrice` is on, the whole wrapper is dropped — and **the stock line lives inside that same
wrapper**, so it disappears too. Verified live on `/knee-rail-calculator`:

```
Knee Rail Fencing with 1.8m Top Rail - Calculator   ← h1
Pay in 3 Payments at 2.5% interest with…            ← finance line, straight after the title
```

No price. No "In Stock". No unit. The title is followed immediately by the finance line.

These are calculators and bespoke/configured items whose price is built entirely from the options the
shopper picks — the base price is meaningless, so it is hidden. Examples: `/knee-rail-calculator`,
`/post-and-rail-fencing-calculator`, `/3200-armco-kit`, `/1600-armco-kit`,
`/150mm-thick-prestressed-concrete-panels`.

**For the mobile app:** this is a real state you must design — a product header with no price *and no stock
indicator*. It is 13× more common than "Free".

---

## 4. "PRICE ON APPLICATION" is not a price state

This is the source of the confusion, and it is worth being blunt about it.

POA is driven **only** by `productType === "POA"`. It has no relationship to `price`.

**Every one of the 33 published POA products has a non-zero stored `price`** (e.g. the EchoGroove acoustic
kits carry stored prices of £513–£1,762). That price is simply never displayed to anyone.

POA products are also routed to a **different Gatsby template** (`price-on-application`), not the normal
product template. Confirmed via the live page-data for `/6-0m-echogroove-reflective-acoustic-fencing-kit`:

```
componentChunkName: component---src-templates-price-on-application-price-on-application-js
```

That template never mounts the price component at all, so a POA product **cannot** display "Free" no matter
what its `price` is. It shows:

```
PRICE ON APPLICATION
Please call 01283 512 111 or email sales@firstfence.co.uk for more information
```

…and no buy controls.

---

## 5. Why `pyramid-barrier-1-metre` behaved the way it did

The product the designer found is explained completely by its record:

```
title:       Pyramid Barrier 1 Metre
sku:         BARR-PLA-4600
productType: POA
status:      TEST
inStock:     false
price:       0        ← irrelevant, never read
```

Three separate things are going on, all of them expected:

1. **Dev-only, not on prod.** `status: "TEST"`. The site build keeps `PUBLISHED` and `TEST` products, then
   strips `TEST` unless the environment variable `ENV_STATE` is `test`. Dev builds with `ENV_STATE=test`,
   production does not. So every `TEST` product exists on dev and on no other environment. This is the
   general answer to *"why can I see this on dev but not prod"* — check `status` first.
2. **It is not even the POA template.** `inStock: false` is checked **before** `productType` when routing,
   so it is built with the **out-of-stock** template. Confirmed via dev page-data:
   `component---src-templates-out-of-stock-product-out-of-stock-product-js`.
3. **The POA message still shows** because the out-of-stock template shares the same header component,
   which prints the POA panel whenever `productType === "POA"`. So an out-of-stock POA product shows the
   POA call/email panel rather than an out-of-stock message.

Its `price: 0` never entered into any of it.

---

## 6. Template routing — the rule the structure docs did not spell out

Page type is decided in this order. **First match wins**, which is why `inStock` beats `POA`:

1. `inStock === false` → **out-of-stock** template
2. `productType === "POA"` → **price-on-application** template
3. `productType` is `SIMPLE` / `CALCULATOR` / `HIRE` → **normal product** template ← the only one that can show "Free"
4. `productType === "GROUP"` → **group product** template
5. anything else → **no page is built at all** (a build warning is logged)

And a product only reaches that routing at all if it passes the build filter: `sku` set, `visible` not
`false`, `status` in `PUBLISHED`/`TEST` (with `TEST` stripped on production), `productType` not null, and a
`path` set.

**Mobile-app consequence:** the app hitting the CDN GraphQL API directly gets **none** of this filtering or
routing. It will receive `DRAFT`, `ARCHIVED` and `TEST` products, invisible products, products with no
path, and products with unroutable `productType` values — all of which the website silently drops. The app
has to reproduce both the filter and the five-way routing itself, in that order.

---

## 7. Data behind the claims

Production Mongo dump, `cdn.products`, 7,885 documents.

**`price` field types:** 6,606 double · 1,238 int · 29 null · 12 missing.

**Zero-or-missing price, by status and type** (top rows): `TEST`/`SIMPLE` 165 · `DRAFT`/`SIMPLE` 76 ·
`PUBLISHED`/`SIMPLE` 50 · `PUBLISHED`/`CALCULATOR` 25 · `ARCHIVED`/`SIMPLE` 24. **Most zero-price products
are test and draft data** — which is exactly why browsing the database gives a misleading impression of how
common the state is on the live site.

**Applying the production build filter** (`sku` set, `visible ≠ false`, `status = PUBLISHED`,
`productType` set, `path` set):

| | Count |
|---|---|
| Zero/missing price, page gets built | 42 |
| …routed to the normal product template (`inStock ≠ false`, type SIMPLE/CALCULATOR/HIRE) | 42 |
| …**and** `ignoreBasePrice ≠ true` → **renders "Free"** | **3** |
| …**and** `ignoreBasePrice = true` → **price block hidden** | 39 |

**POA products:** 33 published, `inStock: true`, visible. **All 33 have `price > 0`.**

**A null or missing `price` behaves identically to `0`** — the price helper parses it, fails, falls back to
`0`, and prints "Free". But no live published product is currently in that state, so "blank price" is a
theoretical state, not an observed one. The structure docs' phrasing ("falls back toward Free/zero") was
right; it just never said no live product does this.

---

## 8. Corrections to make to the structure docs

If and when the originals are revised, these are the edits. Not applied — the originals are unchanged.

- `PRODUCT_PAGE_STRUCTURE.md` line ~356 and `PRODUCT_PAGE_STRUCTURE_SCOPED.md` line ~252 — the **Free**
  row is accurate but needs a live example appended: *"e.g. `/envirorail-bow-top-sample`; only 3 live
  products, all free sample boxes"*. Add that the unit word still renders beside it ("Free EACH").
- Both docs' **"No price / Free"** sections — promote `ignoreBasePrice` from a bullet to the headline case
  (39 live products vs 3), and state explicitly that hiding the price block **also hides the stock line**.
- `PRODUCT_PAGE_STRUCTURE.md` line ~844 — add "POA is driven by `productType`, never by `price`; all 33
  live POA products have a real stored price that is never shown." The current ordering of the bullets
  invites exactly the misreading that prompted this investigation.
- Both docs — add the five-way template routing order from section 6, and the `TEST`-on-dev-only rule.
  Neither is currently stated, and both are needed by anyone reproducing the page in the app.

---

## 9. How to verify any of this yourself

Live template routing for any product, without opening a browser:

```bash
curl -s "https://firstfence.co.uk/page-data/<slug>/page-data.json" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); p=d['result']['data']['product']; \
    print(d['componentChunkName'], p.get('price'), p.get('productType'), p.get('ignoreBasePrice'))"
```

Swap the host for `dev.firstfence.co.uk` to see `TEST`-status products.

Local Mongo (production dump, running in the `firstfence-mongo` container):

```bash
docker exec firstfence-mongo mongosh --quiet \
  "mongodb://root:rootpassword@127.0.0.1:27017/admin" --eval '
  const c = db.getSiblingDB("cdn").products;
  printjson(c.find({
    price: 0, status: "PUBLISHED", visible: {$ne: false},
    sku: {$ne: null}, path: {$nin: [null, ""]},
    inStock: {$ne: false}, ignoreBasePrice: {$ne: true},
    productType: {$in: ["SIMPLE","CALCULATOR","HIRE"]}
  }, {sku:1, title:1, path:1, _id:0}).toArray())'
```

That query is the exact definition of "shows Free on production". It returns three rows.

Note the credentials: the `firstfence`/`password` pair in the `.env` files does not authenticate against
this container — the working pair is the container's own root credentials above.

---

## 10. Source references

- `gatsby-website/gatsby-node.js` ~line 86 — build filter · ~line 189 — `TEST` stripped unless
  `ENV_STATE=test` · ~line 353 — five-way template routing
- `gatsby-website/src/components/price/price.jsx` — `getPriceString(price, { zero: "Free" })`, and the
  `price && price > 0` guard that hides the incl-VAT line
- `gatsby-website/src/utils/general.js` ~line 16 — `getPriceString`; `getFloat` turns null/blank into `0`
- `gatsby-website/src/templates/product/product.js` ~line 384 — `!ignoreBasePrice` wraps price **and**
  `StockMessage` together
- `gatsby-website/src/features/products-utils/product-header/product-header.js` ~line 53 — the POA panel;
  used by both the POA and out-of-stock templates
- `gatsby-website/src/templates/price-on-application/price-on-application.js` — no price component at all
