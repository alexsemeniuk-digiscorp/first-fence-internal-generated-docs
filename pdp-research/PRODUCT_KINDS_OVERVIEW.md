# What kinds of products are there?

A plain-English overview of how one product can differ from another. Written for the question
*"what different kinds of products do we have?"*

Everything here is measured against the live production data (7,885 product records, of which **3,708 are
actually on sale**) and checked against the code that renders them. No existing file has been changed.

---

## The short answer

There **is** a field called `productType` with five values — but it is **not the main way products differ**.
It decides which page layout a product gets, and almost nothing else.

The real differences come from about **ten other things** that can be switched on and off independently.
A product isn't "a type" so much as a **combination of switches**, and most combinations are legal.

So there are two useful answers to the question:

1. **Five official page types** (section 2) — what the system formally calls product types.
2. **Ten ways any product can differ** (section 4) — what actually changes the customer's experience.

And then, most usefully for planning, **the dozen product shapes you'd actually recognise** if you browsed
the site (section 5).

---

## 1. First: "on sale" is itself a difference

Before anything else, a product record may not be a product a customer can see.

| | Count |
|---|---|
| Product records in the database | **7,885** |
| Actually on sale (published, visible, has a code and a web address) | **3,708** |

The rest fall away for four reasons:

- **Status.** Every product is `PUBLISHED`, `DRAFT`, `ARCHIVED` or `TEST`. Only published ones go to the
  live site. `TEST` products *do* appear on the dev site — that is the usual reason something is visible to
  the team but not to customers.
- **Visibility.** A product can be published but marked "not visible" — **1,832 of them are**. These are
  real, priced products that deliberately have no page of their own. They are the building blocks: steel
  lintels sold only inside a kit, and the paid services (PostPro, BreezeGuard, Temporary Works Design) that
  are added from another product's page. **This is a genuine product kind, not an error.**
- **No product code or no web address.** A record with neither can't be sold or linked to.
- **In stock = no.** Marked out of stock, a product gets a stripped-down page with no buy button. Worth
  knowing: **no live product is currently in this state** — the only ones marked out of stock are archived,
  draft or test records. The out-of-stock page exists but nobody sees it today.

---

## 2. The five official product types

| Type | On sale now | What it means |
|---|---|---|
| **SIMPLE** | **3,238** (87%) | An ordinary product. You choose a quantity and buy it. |
| **CALCULATOR** | **227** (6%) | A fencing kit meant to be bought by the metre. |
| **GROUP** | **161** (4%) | A "kit" page listing several separate products you buy together. |
| **HIRE** | **52** (1%) | Rented for a period, not bought. |
| **POA** | **30** (1%) | "Price on application" — no price shown, call us. |

**What the type actually controls:** which of four page layouts the product gets.

| Situation | Page the customer gets |
|---|---|
| Marked out of stock | Cut-down page, no buying |
| Type is POA | "Price on Application" page — no price, no buy button, phone and email |
| Type is SIMPLE, CALCULATOR or HIRE | The normal product page |
| Type is GROUP | The kit page |
| Anything else | **No page is created at all** |

Note the order — **"out of stock" is checked first**, so an out-of-stock POA product gets the out-of-stock
layout (which still shows the POA message, because they share a header).

### The surprise: CALCULATOR is not really a type

Searching the entire website codebase for `CALCULATOR` returns **exactly one result** — the line that
decides page layout, where it is treated identically to SIMPLE. **The type itself changes nothing else.**

What actually makes a product "a calculator" is two separate settings: *show a Metres box* and *bay width*.
And those can be switched on for an ordinary SIMPLE product too — **98 live SIMPLE products have them**, and
they behave exactly like calculators.

**For planning purposes: treat "sold by the metre" as a feature, not a product type.** The type label is
mostly historical.

---

## 3. What each of the five is actually like

**SIMPLE — the ordinary product.** Mesh panels, gates, fixings, lintels, package deals. Quantity box, buy
button. Everything else (options, multibuy pricing, installation) is optional on top.
Example: `/lint-sk90-3750` — a steel lintel, £631.91 each.

**CALCULATOR — the fence kit sold by the metre.** Sold in "bays" (fence sections). The customer types how
many metres of fence they need and the site converts it: **quantity = metres ÷ bay width, rounded up**. So
20 m of fencing with a 2.51 m bay becomes 8 bays. Almost all of them (225 of 227) also have options to
configure, and 121 have multibuy discounts.
Example: `/1-2m-high-868-twin-mesh-security-fencing` — £81.31 per bay, 2.51 m bays.

**GROUP — the kit page.** Instead of one product, the page lists several products with their own quantities
and prices, and adds up a total. Optionally one of them is the "main" product that leads the page. These are
almost never configurable themselves (only 5 of 161 have options) — the products *inside* them are.
Example: `/hdk90wil-steel-lintel` — a lintel range where you pick the length you need from a list.

**HIRE — rented, not bought.** The differences are real and run right through to checkout:
- Priced **per week**, not per item — every price on the page gains "per week".
- Instead of one price there is a **price table**: rows for hire duration, columns for quantity, so longer
  hires and bigger quantities get cheaper rates.
- A **minimum hire term**, in days — in practice **14 days (31 products) or 28 days (37)**.
- The customer picks **start and end dates** on a calendar.
- They can add **purchasable extras** — consumable items bought outright alongside the rental. This is
  almost exclusively a hire feature (27 of the 28 products that have extras are hire products).
- Checkout gains a collection step, because the goods come back.

Example: `/round-top-temp-fencing-hire` — £2.40 each per week, 14-day minimum.

**POA — no price shown.** All 30 are tall acoustic fencing kits where the real price depends on the site.
Worth knowing: **all 30 have a real price stored in the database** (£513–£1,762) that is simply never
displayed. The "price on application" behaviour comes entirely from the type, never from the price.

### Buy/hire twins

**88 products carry a link to their opposite number** — a "Hire Me!" button on a product you can buy, or a
"Purchase Me!" button on one you can hire. These are two separate product records that know about each other,
not one product with two modes. Example: `/police-crowd-control-barrier` links across to
`/hire-police-barrier`.

---

## 4. The ten ways any product can differ

These apply regardless of type, and they are what actually changes the page. Percentages are of the 3,708
products on sale.

### 1. Whether it's configurable
**65% have options to choose** (colour, height, post type, "do you require an end post?"). A product can
have any number of option groups, and each group appears in one of six styles — card grid, colour swatches,
dropdown-ish list, add-on product picker, radio list or tick list. Choosing an option can change the price,
the delivery time, the picture and which other products are offered.

Products with no options at all are just as normal — **35% have none**.

### 2. How the customer says how many they want
Everyone gets a **Quantity** box. On top of that:
- **Metres** — 374 products (10%). Converts to quantity by bay width, rounding up.
- **Area coverage in m²** — **1 product**. Effectively unused.

### 3. How the price is worked out
Several mechanisms stack:
- **A base price** — normal.
- **Option surcharges** — "+ £11.95 each" on a taller panel.
- **Multibuy tiers** — 15% of products get cheaper per unit above certain quantities. Typically 2–3 tiers,
  up to 6. Example: £40 each normally, £32 from 11 up, £29 from 51 up.
- **No base price at all** — 74 products (2%) hide the price entirely because it is built purely from the
  options chosen. Their price *and stock line* disappear together.
- **A "was" price** — 3 products currently show a crossed-out higher price.
- **Free** — 3 products are £0.00 and show the word "Free". All three are free sample boxes.

Everything shown to the customer is **excluding VAT**, with an "incl VAT" line added underneath.

### 4. What unit it's sold in
`EACH` (3,507), `PER BAY` (141), `PER PACK` (42), `KIT` (16), `SAMPLE` (2). Two further units exist in the
system — `SET` and `PALLET` — but **no live product uses them**.

### 5. When the customer gets it
- **In stock** — ships normally.
- **On a lead time** — 47% of products. Shown as "2 Weeks Lead Time" or similar. The wait can also be
  attached to a specific *option*, so picking a colour can push an in-stock product onto a lead time. **727
  products behave this way.**
- **Stock guarantee badge** — 11% carry a "guaranteed in stock" promise, which an option can also remove.
- **Due back in stock date** — adds working days automatically as the date approaches.

### 6. How it gets delivered
- **Delivery multiplier** — a per-product cost weighting. 3,680 products have it at zero (the default);
  28 have a real value between 1.3 and 2.
- **Pallet** — 49 products.
- **Collection only / special** — 164 products.
- **Weight** — set on 412 products.
- **Free delivery** — a flag exists; **no live product uses it**.

### 7. What can be added to it
- **Installation** — 17% of products can be installed by First Fence. Priced either **per metre** or **per
  item**, with extra charges for ground type and site conditions.
- **Upsells** — 61% suggest add-on products right on the page, sometimes with a recommended quantity.
- **Purchasable extras** — hire only, 28 products.
- **Engineering services** — PostPro (63 products), Temporary Works Design (69), BreezeGuard (10). Paid
  third-party reports added from the product page.

### 8. What content it carries
Nearly every product has the basics, but not all:

| Content | Products that have it |
|---|---|
| Images | 100% |
| Specifications | 96% |
| Part of a product range | 97% |
| Downloads (spec sheets, drawings, manuals) | 84% |
| "What's included" list | 63% |
| FAQs | **2%** |
| Custom features | **0.3%** |

FAQs and custom features are rare enough to treat as exceptions.

### 9. Ordering rules
- **Minimum quantity** above 1 — 13 products.
- **Maximum order quantity** — 3 products (the basket silently caps you).
- **Minimum hire term** — hire only.

### 10. Marketing badges
Small coloured labels on the image: **Best Seller** (17 products), **Offer** (69), **Sale** (4), **New** (8).
A fifth, **Buy More Pay Less**, exists but **no live product uses it** — even though 549 products genuinely
have multibuy pricing.

---

## 5. The product shapes you'd actually recognise

This is probably the most useful list. These are the combinations that really occur:

| # | Shape | What it looks like | Roughly how many |
|---|---|---|---|
| 1 | **Plain stock item** | Price, quantity, buy. No options. | ~1,100 |
| 2 | **Configurable item** | Same, plus option groups that change the price | ~2,100 |
| 3 | **Fence kit by the metre** | Metres box, bay width, options, usually multibuy | 374 |
| 4 | **Kit / range page** | A list of related products with individual quantities and a running total | 161 |
| 5 | **Hire item** | Weekly pricing, price table, date picker, minimum term, extras | 52 |
| 6 | **Price on application** | No price, no buy button, phone and email | 30 |
| 7 | **Bespoke (no base price)** | Price hidden until options are chosen | 74 |
| 8 | **Free sample** | £0.00, shows "Free" | 3 |
| 9 | **Hidden component** | Real product, no page — only sold inside a kit | part of the 1,832 |
| 10 | **Hidden service** | PostPro, TWD, BreezeGuard — added from another product's page | part of the 1,832 |
| 11 | **Buy/hire twin** | Ordinary product with a link to its hire version, or vice versa | 88 |
| 12 | **Installable product** | Any of the above, plus a "have it installed" flow | 648 |

Shapes 1–8 are mutually exclusive-ish; 9–12 layer on top.

---

## 6. How to tell which one you're looking at

A quick decision list, in the order the system itself applies:

1. **Is it published and visible?** No → it has no page. It's either not ready, or a deliberate hidden
   component/service.
2. **Is it marked out of stock?** Yes → cut-down page, no buying. (Nothing live is currently in this state.)
3. **Is the type POA?** Yes → no price, call us.
4. **Is the type GROUP?** Yes → kit page listing several products.
5. **Is the type HIRE?** Yes → weekly pricing, dates, minimum term.
6. **Otherwise it's a normal product page** — and what it looks like depends on the switches:
   - Metres box? → sold by the metre
   - Option groups? → configurable
   - Multibuy tiers? → cheaper in bulk
   - Base price hidden? → priced entirely by configuration
   - Installation available? → extra flow before the basket

---

## 7. Things worth knowing before planning

**"Type" is weaker than it looks.** Only HIRE and POA really change behaviour in the code. GROUP swaps the
page layout. CALCULATOR and SIMPLE are the same thing. If we ever rationalise the model, three of the five
values are candidates for removal.

**Almost half the catalogue is hidden.** 1,832 published-but-invisible products is not a data problem — it's
how kits and services are built. Any system that lists "all published products" will be badly wrong.

**Nothing is required.** Every field can be empty. There are live products with no specifications, no
downloads, and no options. Any interface has to cope with a nearly-empty product.

**Several features are switched on in the system but unused in practice** — free delivery, "Buy More Pay
Less", area coverage, the `SET` and `PALLET` units, and the whole out-of-stock page. Don't assume something
works just because the field exists; equally, don't delete them without checking.

**Stock state is not fixed per product.** It changes when the customer picks an option, and it changes on
its own as a due-back date approaches. It cannot be treated as a static property.

**46 live products belong to no category.** They are reachable by direct link and search only.

**Only the website filters the catalogue.** The underlying data service happily returns draft, archived and
test products, hidden products, and products with no page. The website filters them out. **Anything new
built on top of that data — the mobile app, for instance — has to repeat that filtering itself**, in the
same order, or customers will see test data.

---

## 8. Reference: the numbers in one place

Of **3,708 products on sale**:

| | Count | % |
|---|---|---|
| SIMPLE / CALCULATOR / GROUP / HIRE / POA | 3,238 / 227 / 161 / 52 / 30 | 87 / 6 / 4 / 1 / 1 |
| Has options to configure | 2,406 | 65% |
| Suggests upsell products | 2,275 | 61% |
| Has a "what's included" list | 2,334 | 63% |
| Has a lead time | 1,735 | 47% |
| Marked "UK made" | 1,579 | 43% |
| Has multibuy pricing | 549 | 15% |
| Can be installed | 648 | 17% |
| Has a stock guarantee | 415 | 11% |
| Sold by the metre | 374 | 10% |
| Collection only/special | 164 | 4% |
| Has a buy/hire twin | 88 | 2% |
| No base price (bespoke) | 74 | 2% |
| Marked "Offer" | 69 | 2% |
| Pallet delivery | 49 | 1% |
| Hire extras | 28 | 1% |
| Best Seller badge | 17 | 0.5% |
| Minimum quantity above 1 | 13 | 0.4% |
| Free (£0.00) | 3 | 0.1% |
| Has a "was" price | 3 | 0.1% |
| Maximum order quantity | 3 | 0.1% |
| Area coverage input | 1 | — |
| Free delivery · Buy More Pay Less · out of stock | **0** | — |

Also: **1,832** published-but-hidden products, and **46** on-sale products in no category.

---

## 9. Where this came from

**Data:** the production database dump (`cdn` → `products`, `productvariants`, `installationtypes`,
`categories`), 7,885 product records, queried July 2026.

**Code:**
- `gatsby-website/gatsby-node.js` — the catalogue filter and the four-way page-layout decision
- `gatsby-website/src/models/Product.js` — metres-to-quantity conversion, price assembly
- `gatsby-website/src/templates/product/`, `group-product/`, `price-on-application/`,
  `out-of-stock-product/` — the four page layouts
- `gatsby-website/src/utils/product/` — price, lead-time and hire calculations
- `cdn-graphql-v2/src/schema/product.js` — the full field list and the five type values

**Related documents in this folder:** `PRODUCT_PAGE_STRUCTURE.md` (every block on the page),
`PRODUCT_PAGE_STRUCTURE_SCOPED.md` (the buying blocks only), `PRODUCT_PAGE_ZERO_PRICE_FINDINGS.md`
(free vs price-on-application), `LEAD_TIME_BADGE_EXPLAINED.md`, `POSTPRO_BLOCK_EXPLAINED.md`,
`BREEZEGUARD_TWD_BLOCKS_EXPLAINED.md`, `HEIGHT_COMPONENT_QUESTION.md`.
