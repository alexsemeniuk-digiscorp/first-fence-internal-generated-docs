# First Fence Product Page — Structure, Blocks and States

**A reference for the designer redesigning the product page in the ff-uk-mobile React Native app.**

## What this document is

This document describes, in full, what a First Fence product page is made of: every block on the page, the order they appear in, what goes inside each one, when each block shows or hides, and every visual state each block can be in. It is written to be read by a designer who does not read web code.

It is assembled from three sources: the backend that stores product data (`cdn-graphql-v2`), the current web app that renders the page (`gatsby-website`), and a read of four live product pages on firstfence.co.uk. Where those sources disagree or leave something genuinely unknown, this document says so rather than guessing — see **Open questions and gaps** at the end.

**The single most important thing to know before you start:** the product page is not one page. There are four different page types, and each product is assigned exactly one of them when its data is prepared — that assignment never changes while the user browses, so design each type as a fixed variant of the product, not a runtime toggle the app flips. The four are:

- **The full buyable page** — a normal for-sale product (the great majority). This is the page you should design first; the other three are smaller subsets of it.
- **The group / kit page** — a product that is a bundle of other products.
- **The price-on-application page** — a product with no price, showing a "call us" panel instead of a buy button.
- **The out-of-stock page** — a stripped-back page with no price and no buy controls at all.

Almost everything in this document describes the full buyable page. The three thinner pages are covered in their own short sections.

**The second most important thing:** the current web page behaves very differently on a phone-width screen than on a desktop. It is not the same layout simply re-stacked — on mobile the content tabs disappear entirely and are replaced by a stacked list of sections in a different order, and a price/buy bar is pinned to the bottom of the screen. Because you are designing for mobile, the mobile behaviour described throughout is the one that matters most.

---

## The Product data model

This section is a plain-language reference to every piece of data a product can carry. You do not need to read it front to back — use it to look up what a given field means and, crucially, **whether it can be missing**, because missing fields are what drive most of the page's on/off blocks.

### The one rule that governs everything: almost nothing is guaranteed

Every single field on a product can be empty. There is no field the backend forces to have a value — not the title, not the price, not even whether it has a picture. A brand-new product starts almost completely blank and an administrator fills it in over time. Some fields get a starting value the first time a product is created (listed below), but even those can be blank on older products.

The practical consequence for you: **design an empty or absent state for every block.** A block with no data does not error — it simply renders nothing and the blocks below it move up. Roughly a third of the page's blocks work this way.

A related trap: many "show this section" switches and "show this badge" switches are simple yes/no flags, and when nobody has set them they read as *blank*, which the app treats the same as "no". So the normal state of most optional blocks is "off".

### Identity and naming

| Field | Meaning | Can be missing? |
|---|---|---|
| `title` | The product name, shown as the big heading. | Yes (in theory) |
| `sku` | The product code, e.g. `WEB-03393`. Auto-generated if not entered, so usually present. | Rarely |
| `oldSku` | A legacy code, for reference only. | Often |
| `path` | The page's web address slug. If missing, no page is built at all. | Yes |
| `productCategory` | A plain text category name. Note: this is also what the reviews widget is keyed on. | Often |
| `status` | Publishing state — see the allowed values below. New products start as `DRAFT`. | No (defaults) |
| `visible` | A separate on/off switch from `status`. When off (`false`), **no product page is built for the product at all** — the site build excludes it, so it is entirely absent, not merely hidden from category listings. | Yes |
| `metaTitle`, `metaDescription` | Search-engine text only; never shown on the page body. | Often |
| `breadcrumbParent` | An override that pins the product's parent category for the breadcrumb trail. Stored as a bare reference with no readable name attached. | Usually |

**`status` values:** `PUBLISHED`, `DRAFT`, `ARCHIVED`, `TEST`. Only `PUBLISHED` (and `TEST` on staging builds) products get a page. Note the backend itself does not hide draft or archived products when asked directly — the web build is what filters them out.

### The product archetype: `productType`

This one field decides the whole shape of the page. Five values:

| Value | What it means | What comes with it |
|---|---|---|
| `SIMPLE` | A standard buyable product. The default. | The full page. |
| `CALCULATOR` | A product sold via a fencing calculator. | Uses `calculatorLink`, `pricePerMeter`, `showMeters`, `bayWidth`. Still the full page. |
| `HIRE` | A rental product. | Uses `hirePrice`, `hirePrices`, `minHireTerm`, a date picker, and "per week" pricing. Still the full page. |
| `GROUP` | A kit / bundle of other products. | Uses `groupProducts` and the group page. |
| `POA` | Price on application. | No price, no buy button; a phone/email panel instead. Uses the POA page. |

Note that `SIMPLE`, `CALCULATOR` and `HIRE` all use the same full page; the only meaningful question that page ever asks while the user is on it is "is this a hire product?".

### Pricing

There is no single "price" for a product. There are several fields, and the final number shown is calculated on the device from them — **the backend does no price maths at all**, and does not send a VAT-inclusive figure or a VAT rate. The web app multiplies by 1.2 for VAT on the device; the mobile app will have to do the same.

| Field | Meaning | Can be missing? |
|---|---|---|
| `price` | The base price, **excluding VAT**. This is the amount the customer actually pays. | Yes (null on POA) |
| `offerPrice` | **The old "WAS" price**, shown struck through — NOT a discounted price. See the warning below. | Often |
| `costPrice` | Internal cost. Must never be shown to a customer. | Often |
| `pricePerMeter` | Price per metre or foot, used with `showMeters`. | Often |
| `priceUnit` | `METER` or `FOOT` — the unit for `pricePerMeter`. | Often |
| `ignoreBasePrice` | When on, `price` is ignored and the price is built purely from the options the customer picks. | Yes (off) |
| `priceTiers` | The bulk-discount ("multibuy") ladder. A list of `{ quantity threshold, unit price }`. | Often empty |
| `hirePrice` | A flat hire price. | Only on hire |
| `hirePrices` | A rate matrix for hire, by quantity band and week band. | Only on hire |
| `unit` | The noun after the quantity — see the allowed values below. | No (defaults to "each") |
| `freeDelivery` | Free-delivery flag. | Yes (off) |
| `removeFlexiblePayment` | **Inverted:** when ON, it HIDES the finance messaging. | Yes (off) |

> **Warning — `offerPrice` is not a discount.** It is the *old* price, shown crossed out beside the current price. The current price is always `price`. Do not build copy or logic that treats `offerPrice` as the cheaper number — it is the more expensive, historical one.

> **Warning — there is no "From £X" field.** Nothing in the data gives you a price range. If the design wants "From £X" on a configurable product, the app has to compute it by scanning the tiers and option modifiers itself.

Two more numeric fields exist on a product but are **never shown on the page** — `weight` and `deliveryMultiplier`. They feed cart/shipping calculations only, so they have no design impact here; they are listed so the "every field" reference is complete and nobody assumes they were overlooked.

**`unit` values and how the web renders them:** `EA` → "EACH" (the default), `BAY` → "BAY", `PACK` → "PER PACK", `SET` → "SET", `KIT`, `PALLET`, `SAMPLE` → shown as the raw word in capitals. On a hire product the whole thing becomes "per {unit} / per week".

### Per-option price and quantity changes

When a product has configurable options (see Variants below), each option can carry its own adjustments that change the running price and quantity live as the customer picks:

- `priceModifier` — a flat amount added to or subtracted from the price for that option.
- `tierPriceModifier` — replaces the bulk-discount ladder for that option.
- `modifyPriceBy` — named adjustments that apply when another named option is also picked.
- `quantityModifier` / `modifyQtyBy` / `minQuantity` — adjust the quantity or its minimum.

The design consequence: the price block is **live**. It shows a base price, then changes as options are selected.

### Stock and availability

Stock is not one flag. It is several independent fields, and the app decides the message from all of them. The order in which they win is decided on the device, not in the backend.

| Field | Meaning | Can be missing? |
|---|---|---|
| `inStock` | The current source of truth for availability. | Yes — and blank does NOT mean "out of stock" |
| `notInStock` | **Deprecated but still filled in on older products.** Opposite polarity to `inStock`. Avoid; a genuine footgun. | Often |
| `inStockGuaranteed` | Drives the "In Stock Guaranteed" badge. | Yes (off) |
| `leadTime` | Whether a lead time applies at all. | Yes |
| `leadTimeDuration` | The lead time length, in working days. | Yes |
| `dateDueBackInStock` | A future date used to extend the lead time. | Usually |
| `collectionSpecial` | Collection-only flag. | Yes |

An individual option can override stock: it can carry its own `stockMessage` (free text), its own `leadTimeDuration`, and an `excludeFromStockGuaranteed` flag that removes the guarantee badge. So picking a variant can change the stock line, the lead time, and the guarantee badge.

> **There is no stock count.** `inStock` is a yes/no, not a number. You can never show "only 3 left".

### Media

| Field | Meaning | Notes |
|---|---|---|
| `images` | The gallery. A list of image or video entries. | Can be empty or absent — see the no-image note below. |
| `logoCollection` | A strip of brand / accreditation logos. | Almost always present but almost always empty. |

Each gallery entry (`Images`) has: a picture address (`url`), a type (`image` or `video` — note lowercase), a video embed address (`embedUrl`, used only for videos), a `isMain` "hero" flag, `alt` text (often blank), a `caption` (often blank), a `sortOrder` number, and a set of variant tags that link the image to specific options (so the gallery can jump to the right picture when a colour is chosen).

Things to know:
- **The hero flag is not reliable.** Zero, one, or several images can be flagged as the hero. The app falls back to the first image.
- **The app must sort the images itself.** The backend returns them in storage order, not by `sortOrder`.
- **Hidden images are already removed** before they reach the app.
- **No image is a normal state.** A new product has an empty image set. The web app substitutes a hard-coded "no image" placeholder tile so the gallery is never truly empty.

The logo strip (`logoCollection`) is a separate, simpler kind of image with an optional clickable `link`. Because every product is created with an empty logo collection, the strip's default state is "present but empty", and it hides itself unless there is at least one logo.

### Variants and options — the configurator

This is the most complex part of the data. A product can have several **variant groups** (`productVariants`), each a question the customer answers, and each with a list of **options**.

Each variant group has:
- `displayedName` — the customer-facing question, e.g. "Do you require end post?". This is the group heading. (Falls back to `variantName` if blank.)
- `subtitle` — optional helper text under the heading.
- `variantType` — the control type. Six values, each a different control (see the table below).
- `multiselect` — allow more than one option (only meaningful for the PRODUCT type).
- `qtyInput` — show a per-option quantity stepper.
- `showCustomColorPicker` — show a free colour picker alongside swatches.

**`variantType` — the six control types:**

| Value | Control |
|---|---|
| `RADIO` | Single-choice list |
| `CHECKBOX` | Multi-select list |
| `SWATCH` | Colour squares (uses each option's `color`) |
| `SWITCH` | On/off tiles |
| `SELECT` | Dropdown |
| `PRODUCT` | Each option is itself another product, shown as a mini product row. The row can also carry the linked product's `shoppingDesc` (short marketing description) as a tooltip/description. |

Any other value renders nothing at all.

Each option has: a label (`name`), a colour (`color`, for swatches), the price and quantity modifiers listed earlier, a per-option `stockMessage` and `leadTimeDuration`, an `excludeFromStockGuaranteed` flag, a `hideStandard` flag (hides the "Standard" price label when the modifier is zero), an `incompatibleWith` list (option ids this option cannot be combined with — in PRODUCT-type groups it removes the option from the list; otherwise, if the customer picks two clashing options it drives the error/restricted state — see block 20), a `productLink` (the linked product, for the PRODUCT type), a `shoppingDesc` (a short marketing description shown as a tooltip/description on PRODUCT-type option rows — see block 20), and its own set of upsells (`globalUpsellProducts`) that appear only when this option is chosen.

> **Important behaviour: most variant groups start with a selection already made.** The web app auto-selects the first option of every group *except* checkbox groups. So "nothing selected yet" is a state that only exists for checkbox groups; swatches, dropdowns, radios, switches and product-pickers always have something chosen on arrival.

### The name/value content lists

Three separate list-shaped blocks feed the detail sections, and they have different shapes:

| Field | Shape | Feeds |
|---|---|---|
| `moreInformation` | name + value pairs | The specifications table |
| `components` | name + quantity | The "What's Included" table |
| `customFeatures` | a heading + a bullet list | A titled bullet block (not confirmed on the live product page — see Open questions) |
| `tags` | name only | Not shown as content, but tag names trigger some promo banners |
| `filters` | name + value | Category-listing facets, **not** product-page content |

### Documents

`downloads` is a list of downloadable files, each with a display `name`, a file address (`link`, which can be blank if no file was attached), and a sort number. There is no file size or file type in the data, so you cannot show a "PDF, 2MB" style label derived from data.

A `documentType` field also exists in the backend (values `PRODUCT_SPEC_SHEET`, `PRODUCT_DRAWING`, `OPERATIONS_AND_MAINTENANCE_MANUAL`, and `OTHER`), but **the product page does not fetch or use it** — the page's download query only asks for `name`, `link` and sort, and renders one flat list (block 28). It is available if a redesign wants to group downloads by type, but nothing does today.

### Related products and cross-sells

Six separate "other products" relationships exist. Each is a list that can be empty, and each silently drops any product that has been deleted — so a set the admin sized at six can arrive with four, or zero.

| Field | Meaning |
|---|---|
| `relatedProducts` | The "You May Also Like" carousel. |
| `groupProducts` | The children of a group/kit product. |
| `purchasableExtras` | Add-on products, shown on hire products. |
| `upsellProducts` | "Frequently Bought Together". Each carries a recommended quantity and a "charge once" flag, and can be hidden depending on which option the customer picked. |
| `hireProducts` | Hire bundles. |
| Option `productLink` / `globalUpsellProducts` | Products reached through a variant option. |

### FAQs, help articles, installation, and other structured blocks

- `FAQGroups` — groups of question/answer pairs. Each group and each question carries its own publishing status, which the app must filter itself.
- `helpCentreArticles` — a carousel of help-centre article cards.
- `installationType` — a structured installation-pricing configuration (ground types, surcharges, price bands, a minimum charge). Present only on products that offer installation. Its `unit` is the `InstallationTypeUnit` enum with two allowed values — **METERS** or **QUANTITY** — and this is the unit shown on the loaded installation block (block 21). Its surcharge options each carry a `SurchargeUnit` enum — **PERCENTAGE**, **PRICE_PER_METER**, or **PRICE** — which only matters inside the (currently deferred) installation modal.
- `productRange` — just a range name.

### Flags — the 20-odd yes/no switches

There is no structured "badge" object. Merchandising badges and section switches are all independent yes/no flags, all of which read as "off" when blank, and **none of which are mutually exclusive** — several badges can be on at once, and contradictory combinations are possible (e.g. "Sale" on with no old price). You need a stacking order that survives all-flags-on.

Badge-like: `bestSeller`, `isSale`, `isNewProduct`, `offer`, `isBuyMorePayLess`, `freeDelivery`, `uk`, `pallet`, `collectionSpecial`.

Section switches — each turns a specific block on or off:

| Flag | What it switches | Block |
|---|---|---|
| `showMeters` | Shows the Meters input and per-metre pricing | 17 |
| `showAreaCoverage` | Shows the Area Coverage (m²) input and its advisory note | 17, 18 |
| `showPostPro` | Shows the PostPro calculation service | 19 |
| `showBreezeGuard` | Shows the BreezeGuard wind-load service checkbox | 24 |
| `showTempWorksDesign` | Shows the Temporary Works Design service checkbox | 24 |
| `ignoreBasePrice` | **Hides** the whole price + stock line; price is built purely from picked options | 3 |
| `removeFlexiblePayment` | **Inverted — hides** the finance line when on | 6 |
| `enableGroupImages` | Group page only: controls how the kit's images behave | group page |
| `groupHasMain` | Group page only: shows a price + quantity for the designated main child | group page |
| `isMainGroupProduct` | Group page only: marks a child as the designated main product | group page |

> **Dead flag — do not design for it.** `showGatedraftPro` is stored and fetched but nothing on any page reads it (a full search of the web app finds zero uses outside the data query itself). There is no "GatedraftPro" section to build. Treat it like the deprecated `notInStock` field: present in the data, never rendered.

> Note: several badge flags (`bestSeller`, `isSale`, `isNewProduct`, `isBuyMorePayLess`) are read by the page but only actually rendered as an overlay on the first gallery image — see the gallery block. They are mainly used on category listing cards, not on the product page body.

### What does not exist in this data at all

So you do not design blocks the platform cannot fill:

- **No reviews, ratings, or star scores.** The reviews widget on the current site is a third-party script, not backend data.
- **No stock counts** ("only N left").
- **No currency** — every price is a bare number, GBP assumed.
- **No VAT split** in the data — the ×1.2 is done on the device.
- **No delivery estimates or dates** — only a flat delivery note.
- **No block ordering or layout information** — the order in this document comes entirely from the web app, not the data.

---

## Page structure at a glance

Below is the full buyable page, top to bottom. "Always" means the block is on every buyable page; "Conditional" means it appears only under the stated condition.

```
┌─────────────────────────────────────────────┐
│ 1  Breadcrumb trail              (always)     │
│ 2  Header: SKU, title, Buy/Hire  (always)     │
│ 3  Price + stock line            (cond.)      │
│ 4  Hire prices table             (hire only)  │
│ 5  In-stock guarantee badge      (cond.)      │
│ 6  Flexible-finance line         (cond.)      │
│ 7  Image gallery                 (always)     │
│      └ badges overlay on 1st image (cond.)    │
│ 8  Image disclaimer              (always)     │
│ 9  Jump-to-section buttons       (up to 3)    │
│ 10 Logo / accreditation strip    (cond.)      │
│ 11 Trade Installer banner        (cond./tag)  │
│ 12 TWD banner                    (cond./tag)  │
│ 13 MUGA Play banner              (cond./sku)  │
│ 14 Reviews widget (3rd-party)    (always*)    │
│ 15 Multibuy price tiers          (cond.)      │
│ 16 Hire date picker              (hire only)  │
│ 17 Quantity / meters / area      (always qty) │
│ 18 Area-coverage note            (cond.)      │
│ 19 PostPro service               (cond.)      │
│ 20 Variants / configurator       (cond.)      │
│      └ per-option qty stepper    (cond.)      │
│ 21 Installation                  (cond.)      │
│ 22 Frequently Bought Together    (cond.)      │
│ 23 Purchasable extras            (hire only)  │
│ 24 BreezeGuard / TWD service     (cond.)      │
│ 25 Add To Basket box (in-page)   (always)     │
│ 26 Phone-help line               (always)     │
│ 27 Calculator link button        (cond.)      │
│ 28 Detail sections               (always)     │
│      desktop: tabs / mobile: stacked          │
│      • Product Description                     │
│      • Specifications & Downloads              │
│      • What's Included                         │
│ 29 Contact Us banner             (always)     │
│ 30 Related products carousel     (always box) │
│ 31 Help centre articles carousel (cond.)      │
│ 32 Sticky Add To Basket bar      (always)     │
│ 33 FAQ accordion                 (cond.)      │
│ 34 Cart confirmation modal       (on add)     │
└─────────────────────────────────────────────┘
```

\* The reviews widget always renders an empty box, but whether it fills with anything depends on a third-party script.

A note on where things sit: on desktop the page is two columns — the gallery (blocks 7–14) is pinned on the left while the right column (blocks 2–6 and 15–27) scrolls. On mobile everything stacks into one column in the order shown above. Because you are designing for mobile, treat the numbered order above as the real order.

---

## Block by block

Each block below leads with what the user sees and why it exists, then lists when it shows, its states, and notes for the port. Data fields are named in `backticks`.

> **No hover states survive the move to touch.** Several web behaviours below are triggered by hovering a mouse — the "View Restrictions" tooltip on a variant option, the installation ground-type tooltip, the option description on product-type rows. A phone has no hover, so every one of these must be re-specified as a tap target: either a tapped info icon that opens a small popover, a long-press, or (simplest) text made always-visible inline. Each affected block below names the touch trigger to use; where it does not, default to an always-visible inline label.

### 1. Breadcrumb trail

**What it is and why.** The category path from the home page down to this product, so the user knows where they are and can step back up. The final crumb is the product title and is not tappable.

**When shown.** Always, on all four page types.

**Content.** A trail of category names ending in `title`. The trail is worked out ahead of time (when the product's data is prepared), by finding the shortest path through the category tree (or using `breadcrumbParent` if the admin pinned one). It arrives ready-made with the product, so it never loads separately and never fails.

**States.**

| State | Trigger | Appearance |
|---|---|---|
| Full trail | Product sits in at least one category | Home > … > Product title |
| Title only | Product is in no category | Home > Product title (or just the title) |

Trail depth varies a lot in practice — the live hire product had a two-level trail; a security fencing kit had four.

**Note for the port.** Because it is precomputed, the trail the product shows is not necessarily the category the user actually navigated in from. There is no back/next-product navigation.

### 2. Product header — SKU, title, Buy/Hire switch

**What it is and why.** The identity block: the product code, its name, and (sometimes) a link to the opposite hire/buy version of the same product.

**When shown.** Always.

**Content.**
- SKU line: `SKU: {sku}`, shown small above the title. Note the web renders the literal "SKU: " even when the code is blank — so an empty SKU shows a dangling label. On the POA and out-of-stock pages the SKU line hides itself when blank instead.
- The product name (`title`) as the main heading.
- The Buy/Hire switch button (see states).

**States of the Buy/Hire switch.**

| State | Trigger | Appearance |
|---|---|---|
| Hidden | `buyLink` is blank (the common case) | No button |
| "Hire Me!" | `buyLink` set and this is not a hire product | Calendar icon + "Hire Me!", links to the hire version |
| "Purchase Me!" | `buyLink` set and this is a hire product | Cart icon + "Purchase Me!", links to the buy version |

**Note for the port.** The web header has several dead, commented-out elements you should NOT port: compare / wishlist / share icons, and a star-rating row. They are not backed by any data.

### 3. Price and stock line

**What it is and why.** The headline price and the availability line, sitting together directly under the header.

**When shown.** Shown unless `ignoreBasePrice` is on, in which case the whole row (price and stock together) is hidden — used for products priced entirely by configuration. Absent entirely on the POA and out-of-stock pages.

**The price sub-block states.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | `price` greater than zero | Big ex-VAT figure + unit word, then a smaller "£X incl VAT" line beneath (the ×1.2 figure) |
| Free | `price` is zero | The literal word "Free"; the incl-VAT line is hidden |
| No price | `price` is blank | Falls back toward zero / Free; effectively no real price |
| With old price | `offerPrice` is set | A "WAS £X" element appears beside the current price (this is the only discount-style treatment on the page) |

Live examples of the normal state: "£158.39 EACH" / "£190.07 incl VAT"; "£6.81 EACH" / "£8.17 incl VAT"; on hire "£2.50 per each / per week" / "£3.00 incl VAT". Note the incl-VAT line never carries "per week" even on hire — a small inconsistency worth fixing rather than reproducing.

**The unit word.** Driven by `unit` (see the data model for the mapping). On a hire product it becomes "per {unit} / per week".

**The stock sub-block — only two states.** Despite there being many stock fields, this line only ever shows one of two things, both with a small checkbox icon:

| State | Trigger | Appearance |
|---|---|---|
| In stock | Computed lead time is zero | "In Stock" |
| Lead time | Computed lead time is above zero | A lead-time message (see below) |

There is no "out of stock" state on this line, because out-of-stock products are routed to a different page entirely.

**How the lead-time message reads.** The lead time is computed: it starts from the product's own `leadTimeDuration`, takes the longest across all currently-selected options, then adds working days up to `dateDueBackInStock` if that is in the future. So **the stock line can change when the customer picks an option.** The wording ladder (working days, 5 days = 1 week):
- 1 day → "1 Working Day Lead Time"
- 2–4, or any non-week multiple → "{n} Working Days Lead Time"
- exactly 5 → "1 Week Lead Time"
- multiples of 5 → "{n} Weeks Lead Time"
- 0 / blank duration → "Available with lead time" — the plain fallback the shared lead-time wording uses when a lead time is flagged but no number of days is set. Note this text does **not** appear on this block-3 stock line (the line only switches to a lead-time message when the computed duration is above zero); it only surfaces in the badges overlay (block 7a), which can flag a lead time on the boolean `leadTime` flag alone. Design a chip for it anyway.

**Note for the port.** The price shown here is the live, calculated per-unit price once the configurator has loaded; before that it is the raw stored price. See **Loading / first paint** in the States cheat sheet.

### 4. Hire prices table

**What it is and why.** A rate matrix for hire products showing the unit price at different quantities and hire lengths.

**When shown.** Only on hire products; returns nothing on everything else.

**Content and states.**

| State | Trigger | Appearance |
|---|---|---|
| Collapsed (default) | Always the starting state | A single tappable line: "Show Hire Qty/Length Discounts" |
| Expanded | User taps the toggle | A grid: columns are quantity bands (e.g. "1 - 4 BAYS", last band gets a "+"), rows are week bands (first row marked "(minimum term)", last row a "+"), cells are prices |

The table is built from `hirePrices`. The number of rows and columns varies per product. **On a phone this grid can be wider than the screen** — decide its mobile overflow: let it scroll sideways inside its own box, or restructure it (e.g. one quantity band at a time). Do not assume it fits the screen width.

### 5. In-stock guarantee badge

**What it is and why.** A small "In Stock Guaranteed" reassurance badge — a round seal-style graphic reading *In Stock Guaranteed*. Tapping it opens the guarantee's terms-and-conditions PDF. (On the web it is a fixed roughly 80×90px seal image; recreate the wording and seal treatment natively rather than reusing the pixel asset.)

**When shown.** Only when all three of these hold: the computed lead time is zero, no currently-selected option carries `excludeFromStockGuaranteed`, and `inStockGuaranteed` is on. Because it depends on selected options and on the computed lead time, **it can appear and disappear live as the customer configures the product**, and it cannot appear on the very first load — it needs the configurator to have loaded.

**States.** Present, or absent (returns nothing, no reserved space).

### 6. Flexible-finance line

**What it is and why.** A one-line "pay in three instalments" message with the **Iwoca** finance-partner logo (Iwoca is the third-party lender). The logo is tappable and opens the flexible-finance-options page.

**When shown.** Shown unless `removeFlexiblePayment` is on. There is **no minimum price** — the live site showed it on a £2.50/week hire ("three monthly payments of £1.03"), which reads oddly but is what ships. Confirm whether it should be gated by price in the app.

**States.**

| State | Trigger | Appearance |
|---|---|---|
| With amount | A total price is known | "Or three monthly payments of £X with" + the Iwoca logo |
| Without amount | No total price yet | "Pay in 3 Payments at 2.5% interest with" + the Iwoca logo |

Always carries the subtitle "Available to UK Limited businesses only".

### 7. Image gallery

**What it is and why.** The main product imagery: a large image with a thumbnail strip, tap-to-zoom, and video support.

**When shown.** Always (the gallery block always occupies its space). It is never empty — if there are no images a hard-coded "no image" placeholder tile is shown instead. On the live site, single-image products were common, so **the gallery must look right with exactly one image** — that is the normal case, not an edge case.

**Content and states.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | One or more images | Large hero image + thumbnail strip |
| No image | Image set empty or absent | A single "no image" placeholder tile |
| Broken image | An image fails to load | Swaps to the same placeholder |
| Video entry | An entry's type is `video` | A video embed (YouTube/Vimeo) instead of a photo; its thumbnail shows a play icon |
| Pre-initialised | Briefly, before the carousel's controls are ready | Only the first slide shows; the arrows and zoom button are not yet present |

Images are shown sorted by `sortOrder` with the hero image moved to the front. Each image can carry a `caption` line shown beneath it (hidden when blank).

**Interactions (as they work on the current web page).**
- Tap a thumbnail to change the main image.
- Tap the zoom button to open a full-screen zoomable view (videos are excluded from this view).
- **Variant-linked images:** when the customer selects an option, the gallery jumps to the image tagged with that option. This is how "the picture changes when you pick a colour" works. The live gate product swapped its hero image between colour finishes this way.

**Mobile touch model — decide this deliberately, the web model does not carry over.** The web gallery is a thumbnail-strip-plus-arrows-plus-zoom-button design built for a mouse. A phone gallery needs a touch model stated explicitly, and the redesign should choose:
- **Changing image:** swipe left/right between images (the expected phone pattern) versus keeping a tappable thumbnail strip. If swipe is the primary gesture, decide whether to keep the thumbnail strip at all or replace it with page dots.
- **Position indicator:** a row of page dots under the image is the usual phone equivalent of the thumbnail strip.
- **Zoom:** pinch-to-zoom on the image in place (the expected phone gesture) versus a separate tap-to-open full-screen zoom button. Pick one; don't assume the web's button.
- **Variant jump vs swipe position:** when a variant selection jumps the gallery to a tagged image, define what happens to the current swipe position — the gallery should move to and settle on the tagged image (and its dot), so a later swipe continues from there.

**Note for the port.** Thumbnail count is a phone decision, not a desktop one: pick a fixed small number that fits a phone width (roughly 4 thumbnails in a row, or drop the strip in favour of page dots). Ignore the web's desktop breakpoint (it measured 6 thumbnails above 800px wide and 4 below, once on load, and never re-measured — a minor web bug, not a target). The full-screen zoom, arrow, and thumbnail internals were not read in full — see Open questions.

#### 7a. Badges overlay (on the first image only)

**What it is.** A row of small badges laid over the first gallery slide only. Absent on the POA and out-of-stock pages, and absent until the configurator has loaded.

**Content — fixed order, and they stack.** First, exactly one stock badge, then up to four flag badges. Then, independently and in this order, each of these adds a badge when its flag is on: `bestSeller` → "Best Seller", `isSale` → "Sale", `isBuyMorePayLess` → "Buy More Pay Less", `isNewProduct` → "New". All four can show at once — nothing stops them, so the design must survive all badges on.

**The stock badge here is NOT driven identically to the block-3 stock line — design three chip variants:**
- **"In Stock"** — neither a lead time nor the `leadTime` flag applies.
- **A lead-time message** (the same wording ladder as block 3, e.g. "1 Week Lead Time") — when a computed lead-time duration is above zero.
- **"Available with lead time"** — the fallback wording used when a lead time is flagged but no duration number is set.

The trigger difference matters: this badge fires when **either** the standalone boolean `leadTime` flag is on **or** a computed duration exists, whereas the block-3 stock line keys off the computed duration only. So a product with `leadTime` switched on but a computed duration of 0 shows **"Available with lead time" as a badge on the first image while the block-3 line reads "In Stock"** — a legitimate disagreement. Design that combination (an "In Stock" price/stock line alongside a lead-time badge) rather than assuming the two always agree.

### 8. Image disclaimer

**What it is.** A static one-line note directly under the gallery: "Images shown are for illustration purposes only and may differ slightly". No data, no conditions. Always shown.

### 9. Jump-to-section buttons

**What it is and why.** Up to three shortcut buttons that scroll the user down to the detail sections.

**When shown.** Their visibility mirrors the detail sections exactly:

| Button | Shown when |
|---|---|
| "Product Description" | Always |
| "Specifications & Downloads" | `downloads` OR `moreInformation` is non-empty |
| "What's Included" | `components` is non-empty |

**Interaction (mobile).** On a phone, tapping a button smooth-scrolls to the matching stacked section further down the page. (On desktop it switches the detail tab instead — but you are designing mobile, so it is a scroll.)

### 10. Logo / accreditation strip

**What it is and why.** A row of brand and accreditation logos (e.g. LPCB, Secured by Design).

**When shown.** Only when `logoCollection` has at least one logo. Because every product starts with an empty collection, **the default state is hidden** — this block is the exception, not the norm.

**Content and states.** Up to 8 logos (extras are silently dropped). A logo with a `link` is tappable and opens externally; a logo without one is a plain image. On a phone, lay the logos out as a compact row (wrapping to a second row if needed) at the smaller of the web's two logo sizes — the web's "larger above 768px" size is a desktop-only rule and does not apply on a phone. 

**Note for the port.** Unlike the main gallery, hidden logos are NOT filtered out here — if any exist they will show. Watch for that.

### 11. Trade Installer banner

**What it is.** A full-width promotional banner aimed at trade installers — it advertises First Fence's trade/automation installer service and invites the customer to call. Tapping it dials the sales line (01283 512 111).

**When shown.** Only when `tags` contains a tag named exactly `#trade-installer`.

**Note on content.** On the web this is a single flat artwork image (`automation-trade-installer-banner.jpg`) with all its headline and copy baked into the picture — there is no separate text in the data. To redesign it as a native banner you must recreate the message from the artwork: a short "trade installer / automation" pitch plus a call-to-action to phone. Treat the pixel image as reference, not as something to embed.

**States.** Present or absent. Can stack with the TWD banner below.

### 12. TWD banner

**What it is.** A full-width promotional banner for the **Temporary Works Design (TWD)** service. Tapping it opens the internal `/twd` page.

**When shown.** Only when `tags` contains a tag named exactly `#twd`. Independent of the Trade Installer banner — both can appear together.

**Note on content.** As with the Trade Installer banner, the message is baked into a flat artwork image (`twd-banner.jpg`) — recreate the TWD headline and call-to-action natively rather than embedding the picture.

### 13. MUGA Play banner

**What it is.** A full-width banner promoting the **MUGA (Multi-Use Games Area)** custom-kit calculator — a tool for pricing a fenced multi-sport play enclosure. Tapping it opens the MUGA custom-kit calculator page.

**When shown.** Only for six specific products, identified by a hard-coded list of SKUs in the web code (`MESH-KIT-4600` / `4650` / `4700` / `4750` / `4800` / `4850`) — **not** a data flag. Confirm whether this should stay a hard-coded list or move to a flag before porting.

**Note on content.** Again a single flat artwork image (`muga-banner.png`) carries the message; recreate the "build your MUGA kit" pitch and call-to-action natively.

### 14. Reviews widget

**What it is.** An empty box that a third-party reviews script (Feefo) fills in live after the page opens. It requests 3 reviews.

**When shown.** The empty box always renders on the full page (it is the last item in the desktop gallery column; absent on the other three page types). Whether anything appears inside depends entirely on the external script.

**States.** There is no loading, error, or empty state in the web code — if the script is slow or blocked, it is simply blank space, and the page jumps as the reviews pop in later.

> The reviews are keyed on `productCategory`, not on the individual product — so **every product in a category shows the same reviews.** There is no review or rating data anywhere in the First Fence backend; this is entirely external. If the mobile app is to show reviews, the widget spec (or a native equivalent) has to come from outside this system.

### 15. Multibuy price tiers

**What it is and why.** A row of tappable buttons offering bulk-discount prices; tapping one sets the quantity to that threshold.

**When shown.** Only when `priceTiers` is non-empty. Present on some products and not others regardless of complexity — on the live site the cheapest, option-less product (a bag of cement) had one, while the £19,950 configurable gate did not.

**Content and states.** Header "Multibuy Discounts Available". A synthetic "BUY 1+" button at the current price is added in front of the stored tiers, so N stored tiers give N+1 buttons. Each button reads "BUY {n}+" over "£{price} ea". Exactly one is selected at a time — the highest threshold at or below the current quantity. Row counts and thresholds vary wildly (live examples: 16/50/151 and 31/281), so treat it as a variable-length list. **On a phone a long row of buttons will not fit** — decide the mobile behaviour: let the buttons scroll sideways in a single row, or wrap them onto multiple rows.

**Interaction.** Tapping a button sets the quantity, which updates the price everywhere on the page.

### 16. Hire date picker

**What it is and why.** A date-range picker for hire products, with a flexible-hire toggle.

**When shown.** Only on hire products.

**Content and states.** Titled "Choose dates:". A start/end range, an "is flexible" toggle, a derived hire duration, and a minimum hire term. Certain dates are blocked. 

| State | Trigger | Appearance |
|---|---|---|
| No dates chosen | Start or end missing | Blocks the add-to-basket buttons with the error "Please select a date range!" |
| Valid range | Both dates chosen and term met | Shows the duration |

The live hire product also carried a "Need a flexible hire contract?" explanatory toggle and a refundable-deposit notice in this area.

### 17. Quantity / meters / area inputs

**What it is and why.** The number inputs that set how much the customer is buying.

**When shown.** The Quantity field is always present. The other two are conditional.

| Field | Shown when | Behaviour |
|---|---|---|
| Quantity | Always | Snaps up to `minQuantity` (or 1) on blur; can show an error |
| Meters | `showMeters` is on | Snaps to `bayWidth` on blur |
| Area Coverage (m²) | `showAreaCoverage` is on | Snaps to `areaCoverage` on blur |

**States of the Quantity field.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Number with − / + steppers |
| Error | A selected option's minimum quantity exceeds the current quantity | Red border + a message below: "Minimum order quantity for '{option}' is {n}." |

**Note for the port.** There is **no upper limit** on the steppers — `orderLimitQuantity` exists in the data and is sent to the cart but is never enforced or shown. If the app should cap quantity, that has to be designed from scratch. The fields stack vertically on mobile.

**Mobile keyboard.** These fields accept typed numbers, so tapping one raises the on-screen number keyboard. On a phone that keyboard can cover the field (and the bottom sticky buy bar). Specify keyboard-avoidance: the tapped field should scroll into view above the keyboard, and use a numeric keypad.

### 18. Area-coverage advisory note

**What it is.** A small grey paragraph about irregular shapes and the customer's responsibility to confirm quantities.

**When shown.** Only when `showAreaCoverage` is on — it is paired with the area input above.

### 19. PostPro calculation service

**What it is and why.** An add-on engineering-calculations service, offered as a single checkbox with its own add-to-basket button and a file-upload modal.

**When shown.** Only when `showPostPro` is on.

**Content and states.** One checkbox option at a hard-coded price (£1,368.75 in the web source — **not** from data; confirm before baking in). Its own "Add to basket" button opens a file-upload modal. Ticking is a toggle. If nothing is ticked, the button does nothing.

### 20. Variants / configurator

**What it is and why.** The heart of a configurable product: the option groups the customer answers to build their product.

**When shown.** Only when the product has variant groups; returns nothing otherwise. It is absent on the very first load and appears once the configurator has loaded.

**Structure.** One sub-block per variant group, each with a heading (`displayedName`), an optional subtitle, and a set of options rendered as one of the six control types (see the data model). A group renders nothing if it has no visible options.

**Per-option states.** Each option can be in one of four states. Because there are six control types, "selected" and "error" look different in each — give each control a concrete appearance rather than a generic "highlighted":

| State | Trigger | Appearance (suggested visual vocabulary) |
|---|---|---|
| Selected | The option is chosen | Radio/checkbox: filled control + check mark, label emphasised. Swatch: coloured square with a ring/border and a check. Switch: toggle in the on position. Dropdown: the chosen row shown in the closed control. Product row: card with a selected border/tint. |
| Unselected | Not chosen | The plain resting form of the same control (empty radio/checkbox, plain swatch, off switch, unselected row/card). |
| Removed (hidden) | An incompatible option is currently selected — **only in PRODUCT-type groups** | The option is taken out of the list entirely; it does not appear. |
| Error / restricted | The user has selected two options that conflict (reachable mainly in multi-select checkbox groups) | The conflicting option is flagged and add-to-basket is blocked. Appearance: a red-outlined control with an inline message. On checkbox rows the message is inline text "(Not available with {option} Selected)". On product rows the restriction is behind a **"View Restrictions"** label the user must tap to reveal the same text (web shows it on hover — on a phone make it a tap target or show the text inline; see the no-hover note at the top of the block-by-block section). |

**Resolving hidden vs greyed:** the two are different mechanisms, not a contradiction. Incompatibility **removes** an option from the row only in PRODUCT-type groups (the option disappears). The **error/restricted** state is separate — it fires when the customer has actively selected two options that clash (possible when a group allows multiple picks), and shows a restriction message rather than removing anything. There is no greyed-but-present disabled option in the current web app; if the redesign prefers greying-out to removal, that is a new choice to make deliberately. Suggested vocabulary if you do add one: disabled = greyed + non-tappable; error = red outline + message; selected = filled + check.

**The option price label.** To the right of each option: "+ £X (each)" for a positive modifier (or "(each/week)" on hire); the word "Standard" when the modifier is zero; or blank when the option's `hideStandard` is set (and radio/checkbox always render blank at zero). On the live site the price delta and even lead-time warnings were baked right into the option label text, e.g. "Bolt Down +£15.75 (each)" and "Blue RAL 5010 (1 Week Lead Time)".

**Collapse behaviour.** Only swatch (and product-picker) groups truncate to the first 4 options behind a one-way "Show More Options" link. The other control types always show every option. The live site confirmed this on the colour/finish selectors.

**Interactions.** Selecting an option can, live: change the running price, change the stock line and lead time, change the guarantee badge, jump the gallery to that option's image, and add or remove upsells.

**Note for the port.** The mix of controls on one product is real — the live gate had two dropdowns, a radio pair, and a colour swatch set on the same page. Design all six control types.

#### 20a. Per-option quantity stepper

**What it is.** A quantity stepper attached to an individual option.

**When shown.** Only when the option is selected AND its group has `qtyInput` on AND the option's price modifier is not zero.

**States.** Label "Quantity" with − / number / +. Decrement floors at the minimum; increment is unbounded. If a recommended quantity is set, shows "Recommended Quantity: {n}", styled as a warning while the current value is below the recommendation.

### 21. Installation

**What it is and why.** An optional paid installation service, configured through a modal (ground type, surcharges, price bands).

**When shown.** Only when the product has an `installationType` that passes validation; otherwise nothing.

**Three states.**

| State | Trigger | Appearance |
|---|---|---|
| Absent | No installation type, or invalid | Nothing renders |
| Skeleton / disabled | Installation data exists but the configurator has not loaded yet | Title "Installation", an unchecked box, £0, the hard-coded word "METERS", a disabled button |
| Normal | Loaded | Title "Installation", a checkbox, the current price, the lowest available price + unit, and a button opening the configuration modal. While the calculation is incomplete the subtitle "Choose and fill details" shows |

**The unit is data-driven, not always "METERS".** The unit shown next to the lowest price comes from the installation configuration and has two possible values: **METERS** or **QUANTITY** (the `InstallationTypeUnit` enum — see the data model). Only the disabled skeleton state hard-codes the word "METERS"; the loaded/normal state renders whichever unit the data carries, so it can legitimately read "QUANTITY".

**Interaction.** Ticking the box before the calculation is valid force-opens the modal rather than toggling; once valid it toggles directly. The modal covers ground-type selection (with an explanatory tooltip, `groundTypeToolTip`), surcharge options, and price bands with a minimum-charge floor. That ground-type tooltip is hover-triggered on the web — on a phone attach it to a tappable info icon that opens a popover, or show the text inline (see the no-hover note at the top of the block-by-block section).

**Note for the port.** The internal states of the installation modal were not read in full — see Open questions.

### 22. Frequently Bought Together (upsells)

**What it is and why.** A list of recommended add-on products the customer can add alongside the main product.

**When shown.** Only when `upsellProducts` is non-empty.

**Content and states.**

| State | Trigger | Appearance |
|---|---|---|
| Expanded (default) | Always the starting state | Header "Frequently Bought Together" + a "Hide" toggle, then the list |
| Collapsed | User taps "Hide" | Header only |
| Card hidden | The upsell is out of stock, OR a selected option carries "hide this upsell" | That card disappears and its quantity is forced to zero |

Each card: product title, an optional lead-time chip, an image, a quantity stepper (floors at 0, unbounded up), a price with "ea (Ex VAT)", and an optional description (the product's `shoppingDesc`) with a show-more toggle. A recommended-quantity line shows only when the computed recommendation is above zero, and turns to a warning style when the current quantity is below it. The recommendation recomputes live as the parent quantity or the selected options change. Options can also inject additional upsells into this list.

### 23. Purchasable extras

**What it is.** Products bought outright alongside a hire, each its own cart line.

**When shown.** Only on hire products, and only when `purchasableExtras` is non-empty.

**Content.** Title "Purchasable Extras", then one row per extra with a quantity control. Only extras with a quantity above zero are added.

### 24. BreezeGuard / Temporary Works Design service

**What it is and why.** Up to two engineering-calculation add-on services, each a checkbox, sharing one add-to-basket button and a modal.

**When shown.** Only when at least one of `showBreezeGuard` or `showTempWorksDesign` is on. The two flags are independent, so one or both groups can show.

**Content and states.** BreezeGuard: a checkbox "Do you require basic wind load calculation?" with hard-coded copy "£99.00 per design". Temporary Works Design: a checkbox with hard-coded copy "£749.99 per design" plus a note "1 TWD is required per product". **The two checkboxes are mutually exclusive** — ticking one unticks the other; tapping the ticked one clears it. The shared button opens the matching modal; if neither is ticked it does nothing.

**Note for the port.** Both prices are hard-coded in the web source, not from data. Confirm before baking in.

### 25. Add To Basket box (in-page)

**What it is and why.** The primary buy control with the running total.

**When shown.** Always, on the buyable page.

**Content and states.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Big total + the unit word (in capitals), a delivery note, an "£X incl VAT" line, and a full-width "Add To Basket" button |
| Loading | While an add request is in flight — and briefly on first load | The button is disabled (no spinner, it just goes dead) |
| Blocked with error | A validation error exists AND the user has interacted | The first error message shows below the button |

The delivery note here **always** reads "Exact delivery cost added during basket checkout" — including on hire products. This in-page box has no hire variant; the refundable-deposit wording lives only on the sticky bar (block 32).

**Note for the port.** A "Add To Quote List" button here is commented out in the web source — do not port. Errors never show on a fresh, untouched page.

### 26. Phone-help line

**What it is.** A static "Need help or advice?" line with the phone number, linking to the contact page. Always shown, no conditions.

### 27. Calculator link button

**What it is.** A "Need Help Building a Kit?" prompt with a "Take Me to Calculator" button.

**When shown.** Only when `calculatorLink` is set. It is the last item in the buy column.

### 28. Detail sections — Description, Specifications & Downloads, What's Included

**What it is and why.** The long-form product information. **This is the block whose mobile behaviour differs most from desktop, and it is the most important layout fact in this document.**

On desktop these three sections are a tab strip — one visible at a time. **On mobile (at phone width) the tab strip is gone entirely and the three sections are stacked, all expanded at once, and in a different order.** The mobile order is:

1. Specifications & Downloads
2. What's Included
3. Product Description

(Desktop tab order is Description, Specs, What's Included.) Design the mobile stack, not tabs.

Each section self-hides when its data is empty, so the stack can be short. Their conditions:

| Section | Shown when | Content |
|---|---|---|
| Product Description | Always present, but its body is empty if `html` is blank | Rich formatted text authored by the admin — headings, lists, embedded images. Unbounded length, no "show more". Needs a real rich-text renderer. |
| Specifications & Downloads | `moreInformation` OR `downloads` is non-empty | A name/value spec table AND a list of download links, sharing one section. Either half can be empty while the other fills the section. |
| What's Included | `components` is non-empty | A Name/Quantity table |

**Specifications table.** Name/value pairs from `moreInformation`. Live spec tables ranged from 3 rows to 11. Values often carry bracketed qualifiers as part of the text, e.g. "2.0m [Installed]".

**Downloads list.** A flat list of download links — each row is a tappable link (the file address, `link`) showing the document `name` with a download icon. **There is no grouping**: no group headers, no sections by document type — every download sits in one plain list. There is no file size or type available. A row's link can be blank if no file was attached. **The empty downloads state is real** — the live cement product had zero documents but the section still rendered (fed by its specs). Document names are free text with inconsistent casing and can be long, so they must wrap.

**What's Included table.** Name/Quantity rows from `components`. **Key behaviour: each quantity is multiplied by the current order quantity** — so the parts list changes live as the customer changes how many they are buying.

### 29. Contact Us banner

**What it is.** A static "Not sure what to buy?" banner with a phone number, email, and a Contact Us button. Always shown, no conditions.

> **Trap:** the heading "Not sure what to buy? / Discover products to complete your solution" reads like the intro to a product carousel, but it resolves to contact details only — there is no product list here.

### 30. Related products carousel — "You May Also Like"

**What it is and why.** A carousel of related product cards.

**When shown.** The carousel box always renders on the full page, but its contents are fetched live after the page opens (only product references are baked in; the actual cards come from a separate request).

**States — and this block is the worst-behaved on the page.**

| State | Trigger | Appearance |
|---|---|---|
| Code still loading | Briefly, while this section's code is still downloading | A bare "Loading..." text |
| Data loading / empty | While fetching, or if the fetch fails, or if there genuinely are no related products | An empty titled carousel — all three look identical |

There is **no data loading spinner and no error message** — a failed fetch is swallowed and looks exactly like "no related products". The title "You May Also Like" is hard-coded. On a phone, show roughly one card at a time with the next card peeking (about 1.2 cards visible) so it reads as a swipeable carousel; the web's "cards per view respond to width" is a desktop multi-card layout that does not fit a phone.

**Note for the port.** This is a place where you should *add* states the web lacks: a proper loading state and a proper empty/hidden state. None of the four live pages read actually showed this carousel populated — see Open questions.

### 31. Help centre articles carousel

**What it is.** A carousel of help-centre article cards.

**When shown.** Only when `helpCentreArticles` is non-empty. Baked into the page (no live fetch), so no loading state. On a phone, use the same swipeable one-card-with-a-peek layout as the related-products carousel (about 1.2 cards visible), not the desktop multi-card row.

### 32. Sticky Add To Basket bar

**What it is and why.** A buy bar pinned to the bottom of the screen at all times, duplicating the in-page Add To Basket box.

**When shown.** Always, on the buyable page. It floats over the page content.

**Content and states.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Total + the label "TOTAL", an "£X incl VAT" line, a delivery note, and an "Add to Basket" button |

The delivery note here **switches on hire**: it reads "Refundable Hire Deposit and Delivery calculated in the cart" on a hire product, and "Exact delivery cost added during basket checkout" otherwise. (This is the only place the refundable-deposit wording appears — the in-page box at block 25 always shows the standard line.)
| Error | A validation error exists AND the user has interacted | A red card with the first error message appears above the bar |
| Disabled | Error (after interaction) OR loading | Button disabled |

On mobile the delivery note is hidden and the price shrinks and centres.

**Port note — reserve space for it.** Because this bar is pinned over the content, the last blocks on the page (FAQ, carousels) must have enough bottom padding that the bar never covers them, and the bar itself must sit above the device's bottom safe-area inset (the home-indicator strip) so its button stays tappable.

> **This bar duplicates the in-page Add To Basket box (block 25).** The two use different labels — "EACH" at the in-page box, "TOTAL" on the sticky bar — and appear at once. On a phone this redundancy is worth resolving deliberately in the redesign; decide which one the app keeps.

### 33. FAQ accordion

**What it is and why.** A list of expandable question/answer items.

**When shown.** Only when `FAQGroups` is non-empty. Full-bleed, near the bottom of the page.

**Content and states.** Header "Frequently Asked Questions" with a Help Centre link. All groups are flattened into one accordion — **the group names are not shown as separators.** Each item is collapsed by default and expands on tap.

**Note for the port.** The web page does not filter FAQs by their publishing status, so draft FAQs can appear — the app should filter these itself. Group names are fetched but never displayed; decide whether grouping should be shown.

### 34. Cart confirmation modal

**What it is.** An overlay shown after a successful add to basket.

**When shown.** Only after a successful add. A failed add does NOT use this modal — it shows a transient red toast, "Adding of the product failed!", for five seconds.

**Content.** Title "Added to Basket", a green check, and three actions: "Go to basket", "Proceed to checkout", and "Continue Shopping" (which just closes it).

---

## States cheat sheet

This is the section to keep open while designing. Each global state below changes several blocks at once. A state you forget here is a screen you forget to draw.

### Which page type is this?

Decided before the user arrives, from `inStock` and `productType`, in this order:

1. If out of stock → **out-of-stock page** (thinnest; see below). This wins even for a group or configurable product.
2. Else if price-on-application → **POA page**.
3. Else if simple / calculator / hire → **full buyable page** (everything above).
4. Else if group → **group page**.

The three non-buyable pages are described after this cheat sheet.

### Loading / first paint

*("First paint" / "first load" here means the very first moment the page shows, before the interactive configurator has loaded.)*

The page shows content immediately from baked-in data, then the interactive configurator loads a moment later and things recalculate. Expect a visible "settle":

- Price and quantity first show raw stored values, then switch to the calculated values (which include any auto-selected option's price change).
- Blocks that need the configurator — **Variants, per-option steppers, Installation (shows its disabled skeleton), Frequently Bought Together, the badges overlay, the in-stock guarantee badge** — are absent or skeletal on first load, then appear.
- Both Add To Basket buttons are briefly disabled on load.

There is no skeleton UI anywhere in the current web app. **The port should decide deliberately** whether to show the settled values directly (it can, since the data is local) or to design loading states. Two blocks genuinely need loading/empty states you should add: the related-products carousel (block 30) and the reviews widget (block 14).

### In stock vs lead time (there is no "out of stock" on this page)

Every product that reaches the buyable page is either "In Stock" or shows a lead-time message — out-of-stock products go to a different page. This affects:

- **Block 3** (stock line): "In Stock" or a lead-time message. Keys off the computed duration only (lead time shows when duration > 0).
- **Block 7a** (badges overlay): a similar chip, but **not driven identically** — it fires on the boolean `leadTime` flag OR a computed duration, and has a third "Available with lead time" wording for the flag-on-but-no-duration case. So the badge can read a lead time while the block-3 line still reads "In Stock". See block 7a.
- **Block 5** (guarantee badge): only shows when lead time is zero.

Because the lead time is computed from selected options and a due-back date, **selecting an option can flip a product from "In Stock" to a lead time**, changing all three of the above live.

### No price / Free / POA

- **No price** (`price` blank): the price block degrades toward Free/zero.
- **Free** (`price` is zero): shows the word "Free", hides the incl-VAT line.
- **`ignoreBasePrice` on**: the entire price + stock line (block 3) is hidden.
- **POA**: a different page entirely — no price, no buy controls, a call/email panel.

### Old-price / discount

Driven only by `offerPrice`. When set, block 3 shows a "WAS £X" beside the current price. Remember `offerPrice` is the *old, higher* number. No sale badge is tied to it directly (the "Sale" badge is a separate flag). None of the four live pages showed this state — see Open questions.

### No image

A normal, common state. The gallery (block 7) shows a hard-coded placeholder tile; it is never truly empty. The badges overlay still applies to that placeholder.

### Has variants (configurable)

When the product has variant groups (block 20 present):

- Blocks 3 (price), 3 (stock line), 5 (guarantee badge), and 7 (gallery image) all become **live** and can change as options are picked.
- Frequently Bought Together (block 22) can gain or lose cards as options are picked.
- Validation errors become possible (see below), which affect both Add To Basket controls (25 and 32).
- Most groups arrive with a selection already made; only checkbox groups start empty.

### Hire product

When it is a hire product:

- Block 4 (hire prices table) appears.
- Block 16 (date picker) appears.
- Block 23 (purchasable extras) can appear.
- Price shows "per {unit} / per week"; delivery notes change to the refundable-deposit wording.
- A "Purchase Me!" cross-link may appear in the header.

### Validation / cannot add to basket

Three possible errors, of which only the first is ever shown, and only after the user has interacted with the page:

1. "Please check your selected options!" — an incompatible option combination is selected.
2. "Minimum order quantity is {n} for the selected option." — quantity below an option's minimum. (Note the quantity field itself shows a differently-worded version of this same problem — an inconsistency to standardise in the port.)
3. "Please select a date range!" — a hire product with no dates chosen.

When present (after interaction), the error disables both Add To Basket controls and shows above the sticky bar and below the in-page button.

### All-badges-on

Because badge flags are independent and non-exclusive, design for the worst case: the stock chip plus "Best Seller", "Sale", "Buy More Pay Less" and "New" all stacked on the first gallery image at once.

---

## The three non-buyable page types

### Out-of-stock page

The thinnest page. In order: breadcrumb → gallery + disclaimer → header (title, SKU) → Specifications & Downloads → What's Included → Product Description → related products.

It has **no price, no quantity, no variants, no add-to-basket, no sticky bar, no stock line, no badges, no tabs** (the three detail sections are stacked directly). Notably, **it does not say "out of stock" anywhere** — the absence of buying controls is the only signal. The gallery badges overlay is suppressed. Consider whether the mobile app should instead show an explicit "Out of stock" / "Notify me" treatment, which would be new design.

### Price-on-application (POA) page

In order: breadcrumb → header → gallery + disclaimer → detail sections → related products.

The header carries the POA panel: a "PRICE ON APPLICATION" heading plus "Please call 01283 512 111 or email sales@firstfence.co.uk for more information", with tap-to-call and tap-to-email. **No price, no quantity, no variants, no add-to-basket, no sticky bar, no FAQ, no upsells.** This header also appends a special note for a hard-coded list of 30 timber SKUs.

### Group / kit page

A bundle page. It adds a list of the kit's child products with per-child quantity controls. The key switch is `groupHasMain`: when on, the page shows a price block and quantity for the designated main child; **when off, both the price and quantity are hidden** and the page is purely a kit list. The total price is the sum across all children, and the lead time is the longest across all children. It has **no variants, no date picker, no upsells, no reviews, and none of the promo banners.** This document does not cover the group page block-by-block — if the app must support group products, commission a separate read of the group template.

---

## Open questions and gaps

Stated honestly, so the port does not build on guesses.

- **Block order comes from the web app, not the data.** The backend has no concept of block order or layout. The order in this document is read from the current web template. It is accurate for the web, but the redesign is free to reorder — nothing in the data forces this sequence.

- **The mobile section order differs from desktop, possibly by accident.** Mobile stacks Specifications → What's Included → Description; desktop tabs go Description → Specs → What's Included. Whether the mobile order is deliberate (specs first matters more on a phone) or an oversight is unknown. Pick one deliberately.

- **The in-page Add To Basket box and the sticky bar are redundant on a phone.** They show at once with different labels. Decide which the app keeps.

- **No discounted product was observed, and no live page confirmed the "WAS price" treatment.** The code supports it via `offerPrice`, but none of the four live products used it. Confirm the exact appearance before finalising.

- **No out-of-stock, no-image, or POA product was observed live.** The out-of-stock and POA behaviours here come from the code only. In particular, the full out-of-stock experience (does it ever say why?) should be confirmed with the team, and the app may want to design a clearer treatment than the web's silent one.

- **The reviews widget has no First Fence data behind it at all.** It is an external Feefo script keyed on category, so all products in a category share reviews. If mobile is to show reviews, the widget spec must come from elsewhere; treat this as out of scope until sourced.

- **The related-products carousel was never seen populated** across the four live pages read, and its web implementation has no loading or error state. The port should add proper loading and empty/hidden states.

- **`customFeatures` (the titled-bullet block) was not confirmed on the live product page.** It exists in the data and can be fetched, but the block-order read of the web template did not place it on the page. Confirm whether it renders on the product page at all before designing a block for it.

- **Several blocks are hard-coded, not data-driven.** The MUGA banner keys off a hard-coded six-SKU list; the PostPro (£1,368.75), BreezeGuard (£99.00) and TWD (£749.99) prices are hard-coded in the web source; the finance line appears with no price threshold. Confirm each is stable, and whether it should move to data, before baking into the app.

- **Several blocks are UK/business-specific promos** (Trade Installer, TWD, MUGA, BreezeGuard, PostPro, finance line, calculator link). Confirm which are in scope for the mobile port at all — dropping the out-of-scope ones removes roughly seven conditional blocks.

- **Component internals not fully read.** The image gallery's full-screen zoom and its exact rule for matching an option to an image, the installation modal's internal states, and the hire date picker's calendar behaviour were not read to the bottom. If the app replicates them closely, read those components first.

- **Quantity has no upper limit in the web app.** `orderLimitQuantity` is carried to the cart but never enforced or shown. If the app should cap quantity, design it fresh.

- **The deprecated `notInStock` field is still populated on older products** and has the opposite meaning to `inStock`. If the app reads stock directly from the backend, this is a real trap — prefer `inStock`.

- **The lead-time unit is assumed to be working days.** The data does not label the unit; the web app treats it as working days (5 = 1 week). Confirm.

- **Does the mobile app read the backend directly or the web build?** This matters for the draft-content leak: the backend does not filter out draft/archived/test products or draft FAQs, so an app reading it directly must filter by status itself. The web build already filters. Confirm the app's data source.

---

## Appendix: for developers

The files behind each block, so the doc can be traced to source. All paths are relative to `/home/osemeniuk/Documents/Work/Projects/first-fence/`.

**Data model and backend (all in `cdn-graphql-v2/`):**
- Product type and enums: `src/schema/product.js`, `src/schema/index.js`
- Variants: `src/schema/product-variant.js`; images: `src/schema/product-images.js`, `src/schema/image.js`, `src/schema/image-collection.js`
- Installation: `src/schema/installation-type.js`, `src/schema/surcharge.js`; FAQs: `src/schema/faq-group.js`, `src/schema/faq.js`; blog/help articles: `src/schema/blog-post.js`
- Mongoose models: `src/models/product.js`, `src/models/product-variants.js`, `src/models/product-images.js`, `src/models/image.js`
- Resolvers (relation hydration, image URL rewrite/filter, defaults): `src/resolvers/product.js`, `src/resolvers/product-variant.js`, `src/resolvers/product-files.js`
- Create/upload defaults: `src/resolvers/product.js` (createProduct ~54–100, uploadProducts ~245–260); `src/utils/constants.js`
- Fetch entry points and pagination: `src/schema/index.js` (79–102), `src/schema/page-info.js`
- No pricing/stock/review logic exists: `src/services/` contains only `s3.js` and `webhook.js`

**Routing, build, breadcrumbs (all in `gatsby-website/`):**
- Template routing and build filter: `gatsby-node.js` (363–401, 85–103)
- Breadcrumb computation: `gatsby-node.js` (200–361)

**The full product page (all in `gatsby-website/`):**
- Template and block order: `src/templates/product/product.js`; layout CSS: `src/templates/product/product.module.css`
- The interactive model (defaults, auto-selection, field renames, price/qty computation): `src/models/Product.js`, `src/models/ProductVariant.js`
- Price/VAT: `src/components/price/price.jsx`, `src/components/price/unit/unit.js`, `src/utils/general.js`
- Stock and lead time: `src/components/stock-message/stock-message.js`, `src/utils/product/lead-time.js`, `src/components/stock-guarantee-message/stock-guarantee-message.jsx`, `src/utils/product/stock-guaranteed.js`
- Header and buy/hire switch: `src/components/product-header/product-header.js`, `src/features/hire-product/buy-switch/buy-switch.js`
- Finance line: `src/components/iwoca-banner/iwoca-banner.js`
- Gallery: `src/components/image-gallery/image-gallery.js`, `.../fullscreen-gallery/fullscreen-gallery.jsx`, `.../badges/badges.jsx`, `src/utils/product/images.js`, `src/utils/gallery.js`, `src/components/image/image.js`
- Disclaimer, logos, banners, reviews: `src/components/disclaimer/disclaimer.js`, `src/components/product-logo-collection/product-logo-collection.jsx`, `src/features/product/trade-installer-banner/`, `.../twd-banner/`, `.../muga-play-banner/`, `src/features/product/feefo-review/feefo-review.jsx`, `gatsby-ssr.js`
- Shortcut buttons: `src/features/product/product-details-shortcut-buttons/product-details-shortcut-buttons.jsx`
- Multibuy tiers: `src/components/price-tiers/price-tiers.jsx`, `.../price-button/price-button.jsx`
- Hire date picker: `src/components/calendar-date-picker/`, `src/constants/product.js`
- Quantity/meters/area: `src/components/product-inputs/product-inputs.jsx`, `src/components/number-field/number-field.js`
- PostPro: `src/features/post-pro/post-pro.js`
- Variants: `src/features/product/variants/` (variants.js and the per-type option renderers under `swatch/`, `select-box/`, `switch/`, `radio/`, `checkbox/`, `product/`, `quantity-box/`)
- Installation: `src/features/product/installation/Installation.js` and the modal under `.../installation-modal/`
- Upsells / extras: `src/features/product/upsell-products/`, `src/features/hire-product/purchasable-products/`
- BreezeGuard / TWD: `src/features/breezeguard/`
- Add to basket (in-page and sticky): `src/features/product/add-to-basket-button/add-to-basket-button.js`, `src/components/add-to-basket-banner/add-to-basket-banner.js`
- Validation: `src/utils/product/validation.js`, `src/constants/error-codes.js`
- Phone / calculator / contact: `src/components/phone-details/phone-details.jsx`, `src/features/product/calculator-link-button/calculator-link-button.jsx`, `src/components/contact-us-banner/contact-us-banner.jsx`
- Detail sections (tabs + mobile stack): `src/features/product/product-details-tabs/product-details-tabs.jsx`, `src/templates/product/product.js` (mobile sections), and the content components `src/components/product-description/`, `src/components/more-information/`, `src/components/specifications/`, `src/components/downloads/`, `src/components/product-components/`
- Related / help / FAQ / cart modal: `src/features/group-product/related-products/related-products.js`, `src/services/actions/product-actions.js`, `src/components/products-carousel/`, `src/components/help-center-carousel/help-center-carousel.jsx`, `src/components/faq/faq.js`, `src/components/cart-modal/cart-modal.js`

**The three non-buyable templates (all in `gatsby-website/`):**
- Out of stock: `src/templates/out-of-stock-product/out-of-stock-product.js`
- Price on application: `src/templates/price-on-application/price-on-application.js`, `src/features/products-utils/product-header/product-header.js`
- Group: `src/templates/group-product/group-product.js`, `src/models/GroupProduct.js`
