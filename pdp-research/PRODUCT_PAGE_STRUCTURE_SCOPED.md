# First Fence Product Page — Core Product & Buying Blocks (Scoped)

## What this document is

This is a **focused, trimmed version** of `PRODUCT_PAGE_STRUCTURE.md`. It keeps only the blocks that describe the product itself and the controls used to configure and buy it — the price, the stock line, the multibuy discounts, the image gallery, the configurator (Post Type, Pale Type, "Do you require an end post?" and the like), the quantity inputs, installation, "Frequently Bought Together", the add-to-basket controls, and the product-information tables.

It deliberately **leaves out** the marketing and navigation furniture — the promo banners (Trade Installer, TWD, MUGA), the finance line, the third-party reviews widget, the "You May Also Like" and help-centre carousels, the FAQ accordion, and the contact banners. Those are documented in the full file; a short list of what was cut, and why, is at the end.

It is written for a **designer building the ff-uk-mobile React Native app**, who is redesigning this page for a phone. It describes what a shopper sees, when each block appears, and every state each block can be in. Underlying data field names are given in `backticks` so a developer can trace them, but you do not need to read code to use this.

**Where the full document and this one disagree, the full one wins** — this is a subset of it, generated from the same source.

---

## The product data model (scoped)

You do not need to read this front to back. Use it to look up what a field means and — most importantly — **whether it can be missing**, because missing fields are what switch most blocks on and off.

### The one rule that governs everything: almost nothing is guaranteed

Every field on a product can be empty. The backend forces no field to have a value — not the title, not the price, not even whether there is a picture. A new product starts almost blank and an administrator fills it in over time.

The practical consequence: **design an empty/absent state for every block.** A block with no data does not error — it renders nothing and the blocks below it move up. Many "show this section" switches are simple yes/no flags that read as *blank* when nobody set them, and blank is treated as "no" — so the normal state of most optional blocks is "off".

### The product archetype: `productType`

One field decides the overall shape of the page. Five values; the three that use this full buying page are `SIMPLE`, `CALCULATOR`, and `HIRE`.

| Value | Meaning | What comes with it |
|---|---|---|
| `SIMPLE` | Standard buyable product (the default) | The full buying page in this document |
| `CALCULATOR` | Sold via a fencing calculator | Adds `calculatorLink`, `pricePerMeter`, `showMeters`, `bayWidth`; still the full page |
| `HIRE` | Rental product | Adds `hirePrice`, `hirePrices`, a date picker, "per week" pricing; still the full page |
| `GROUP` | A kit/bundle of other products | Uses a separate group page (not covered here) |
| `POA` | Price on application | No price, no buy button (not covered here) |

While the user is on the buying page, the only archetype question it ever asks is "is this a hire product?".

### Pricing

There is **no single "price"**. The final number is calculated on the device — the backend does no price maths and sends no VAT-inclusive figure. The app multiplies by 1.2 for VAT itself.

| Field | Meaning | Can be missing? |
|---|---|---|
| `price` | Base price, **excluding VAT** — the amount the customer actually pays | Yes |
| `offerPrice` | The old **"WAS" price**, shown struck through — **not** a discount | Often |
| `pricePerMeter` | Price per metre/foot, used with `showMeters` | Often |
| `priceUnit` | `METER` or `FOOT` — the unit for `pricePerMeter` | Often |
| `ignoreBasePrice` | When on, `price` is ignored and the price is built purely from picked options | Yes (off) |
| `priceTiers` | The bulk-discount ("multibuy") ladder: a list of `{ quantity threshold, unit price }` | Often empty |
| `hirePrice` / `hirePrices` | Flat hire price / hire rate matrix by quantity and week band | Hire only |
| `unit` | The noun after the quantity (see below) | Defaults to "each" |

> **`offerPrice` is not a discount.** It is the *old, higher* price, shown crossed out next to the current price. The current price is always `price`. Never treat `offerPrice` as the cheaper number.

> **There is no "From £X" field.** Nothing gives you a price range. If the design wants "From £X" on a configurable product, the app must compute it by scanning the tiers and option modifiers itself.

**`unit` values and how they render:** `EA` → "EACH" (default), `BAY` → "BAY", `PACK` → "PER PACK", `SET` → "SET", and `KIT` / `PALLET` / `SAMPLE` → the raw word in capitals. On a hire product the whole thing becomes "per {unit} / per week".

### Per-option price and quantity changes

When a product has configurable options, each option can carry its own adjustments that change the running price and quantity **live** as the customer picks:

- `priceModifier` — a flat amount added to/subtracted from the price for that option.
- `tierPriceModifier` — replaces the bulk-discount ladder for that option.
- `modifyPriceBy` — named adjustments that apply when another named option is also picked.
- `quantityModifier` / `modifyQtyBy` / `minQuantity` — adjust the quantity or its minimum.

The design consequence: **the price block is live** — it shows a base price, then changes as options are selected.

### Stock and availability

Stock is several independent fields; the app decides the message from all of them.

| Field | Meaning | Can be missing? |
|---|---|---|
| `inStock` | Current source of truth for availability | Yes — and blank does **not** mean "out of stock" |
| `notInStock` | **Deprecated** but still filled on older products; opposite polarity to `inStock` — a footgun, avoid | Often |
| `inStockGuaranteed` | Drives the "In Stock Guaranteed" badge | Yes (off) |
| `leadTime` | Whether a lead time applies at all (a yes/no flag) | Yes |
| `leadTimeDuration` | The lead-time length, in working days | Yes |
| `dateDueBackInStock` | A future date that extends the lead time | Usually |

An individual option can override stock: its own `stockMessage` (free text), its own `leadTimeDuration`, and an `excludeFromStockGuaranteed` flag. So **picking a variant can change the stock line, the lead time, and the guarantee badge.**

> **There is no stock count.** `inStock` is a yes/no, never a number — you can never show "only 3 left". Out-of-stock products never reach this page; they route to a separate page.

### Media (the gallery)

`images` is the gallery: a list of image or video entries. It can be empty or absent.

Each entry has a picture address (`url`), a type (`image` or `video`, lowercase), a video embed address (`embedUrl`), an `isMain` "hero" flag, `alt` text (often blank), a `caption` (often blank), a `sortOrder` number, and variant tags that link the image to specific options (so the gallery can jump to the right picture when a colour is chosen).

Things to know:
- **The hero flag is unreliable** — zero, one, or several images can be flagged; the app falls back to the first image.
- **The app must sort the images itself** by `sortOrder` (the backend returns storage order).
- **Hidden images are already removed** before they arrive.
- **No image is a normal state** — a new product has an empty image set; the web substitutes a hard-coded "no image" placeholder tile.

### Variants and options — the configurator

The most complex part of the data. A product can have several **variant groups** (`productVariants`), each a question the customer answers (e.g. "Post Type", "Pale Type", "Do you require an end post?"), each with a list of **options**.

Each variant group has:
- `displayedName` — the customer-facing question / group heading (falls back to `variantName` if blank).
- `subtitle` — optional helper text under the heading.
- `variantType` — the control type (six values, below).
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
| `PRODUCT` | Each option is itself another product, shown as a mini product row (can carry `shoppingDesc` as a description) |

Any other value renders nothing.

Each option has: a label (`name`), a colour (`color`, for swatches), the price/quantity modifiers above, a per-option `stockMessage` and `leadTimeDuration`, an `excludeFromStockGuaranteed` flag, a `hideStandard` flag (hides the "Standard" label when the modifier is zero), an `incompatibleWith` list (option ids this one cannot combine with), a `productLink` (for the PRODUCT type), a `shoppingDesc` (short description on product rows), and its own upsells (`globalUpsellProducts`) that appear only when this option is chosen.

> **Most groups start with a selection already made.** The app auto-selects the first option of every group *except* checkbox groups. So "nothing selected yet" only exists for checkbox groups; swatches, dropdowns, radios, switches and product-pickers always arrive with something chosen.

### The product-information lists

Three list-shaped fields feed the detail tables:

| Field | Shape | Feeds |
|---|---|---|
| `moreInformation` | name + value pairs | The Specifications table |
| `components` | name + quantity | The "What's Included" table |
| `downloads` | display name + file link (+ sort) | The Downloads list |

`downloads` files have **no size or type** in the data, so you cannot show a "PDF, 2MB" label. A download's `link` can be blank if no file was attached.

### Add-on products

| Field | Meaning |
|---|---|
| `upsellProducts` | "Frequently Bought Together". Each carries a recommended quantity and a "charge once" flag, and can be hidden depending on which option was picked. |
| `purchasableExtras` | Add-on products bought outright, shown on hire products. |
| Option `globalUpsellProducts` | Extra upsells injected into "Frequently Bought Together" when a specific option is chosen. |

Each of these is a list that can be empty, and each **silently drops any product that has been deleted** — so a set of six can arrive as four, or zero.

### Installation

`installationType` is a structured installation-pricing configuration (ground types, surcharges, price bands, a minimum charge). Present only on products that offer installation. Its `unit` has two possible values — **METERS** or **QUANTITY** — and this is the unit shown on the loaded installation block.

### The section-switch flags that matter here

Independent yes/no flags, each reading "off" when blank:

| Flag | What it switches on |
|---|---|
| `showMeters` | The Meters input and per-metre pricing |
| `showAreaCoverage` | The Area Coverage (m²) input and its advisory note |
| `ignoreBasePrice` | **Hides** the whole price + stock line; price is built purely from picked options |
| `priceTiers` non-empty | The Multibuy discounts row |

### What does not exist in this data at all

So you do not design blocks the platform cannot fill:

- **No stock counts** ("only N left").
- **No currency** — every price is a bare number, GBP assumed.
- **No VAT split** — the ×1.2 is done on the device.
- **No delivery estimates or dates** — only a flat delivery note.
- **No price range / "From £X"** — must be computed if wanted.

---

## Page structure at a glance (scoped)

The blocks in scope, in the order they stack **on a phone** (the mobile order is the real one — on desktop the page is two columns, but you are designing mobile). "Always" = on every buyable product; "Conditional" = only under the stated condition.

```
┌──────────────────────────────────────────────────┐
│ A  Header: SKU, title, Buy/Hire      (always)      │
│ B  Price + stock line                (cond.)       │
│ C  In-stock guarantee badge          (cond.)       │
│ D  Image gallery                     (always)      │
│      └ badges overlay on 1st image   (cond.)       │
│ E  Multibuy discounts                (cond.)       │
│ F  Hire date picker                  (hire only)   │
│ G  Quantity / Meters / Area inputs   (qty always)  │
│      └ area-coverage advisory note   (cond.)       │
│ H  Variants / configurator           (cond.)       │
│      └ per-option quantity stepper   (cond.)       │
│ I  Installation                      (cond.)       │
│ J  Frequently Bought Together        (cond.)       │
│ K  Purchasable extras                (hire only)   │
│ L  Add-on service toggles            (cond.)       │
│      (PostPro / BreezeGuard / TWD)                 │
│ M  Add To Basket box (in-page)       (always)      │
│ N  Detail sections                   (always)      │
│      • Specifications & Downloads                  │
│      • What's Included                             │
│      • Product Description                          │
│ O  Sticky Add To Basket bar          (always)      │
│ P  Cart confirmation modal           (on add)      │
└──────────────────────────────────────────────────┘
```

On desktop the gallery is pinned left while the right column scrolls; on mobile everything stacks into the single column above. Note the **detail-section order flips on mobile** (Specifications first, Description last) — see block N.

---

## Block by block

Each block leads with what the user sees, then when it shows and every state it can be in.

> **No hover states survive the move to touch.** A few web behaviours here are triggered by hovering a mouse — the "View Restrictions" tooltip on a variant option, the installation ground-type tooltip, the description on product-type option rows. A phone has no hover, so re-specify each as a tap target: a tapped info icon that opens a small popover, a long-press, or (simplest) always-visible inline text.

### A. Product header — SKU, title, Buy/Hire switch

**What it is.** The identity block: the product code, its name, and sometimes a link to the opposite hire/buy version of the same product.

**When shown.** Always.

**Content.**
- SKU line: `SKU: {sku}`, small, above the title. The web renders the literal "SKU: " even when the code is blank, leaving a dangling label — worth fixing in the port.
- The product name (`title`) as the main heading.
- The Buy/Hire switch button (states below).

**States of the Buy/Hire switch.**

| State | Trigger | Appearance |
|---|---|---|
| Hidden | `buyLink` blank (the common case) | No button |
| "Hire Me!" | `buyLink` set, not a hire product | Calendar icon + "Hire Me!", links to the hire version |
| "Purchase Me!" | `buyLink` set, is a hire product | Cart icon + "Purchase Me!", links to the buy version |

**Port note.** The web header has dead, commented-out compare / wishlist / share icons and a star-rating row — **do not port them**; no data backs them.

### B. Price and stock line

**What it is.** The headline price and the availability line, directly under the header.

**When shown.** Shown unless `ignoreBasePrice` is on, in which case the whole row (price and stock together) is hidden — used for products priced entirely by configuration.

**Price sub-block states.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | `price` > 0 | Big ex-VAT figure + unit word, then a smaller "£X incl VAT" line (the ×1.2 figure) |
| Free | `price` is 0 | The word "Free"; the incl-VAT line is hidden |
| No price | `price` blank | Falls back toward Free / zero |
| With old price | `offerPrice` set | A "WAS £X" element beside the current price (the only discount-style treatment on the page) |

Live examples: "£158.39 EACH" / "£190.07 incl VAT"; on hire "£2.50 per each / per week". The unit word is driven by `unit` (see the data model).

**Stock sub-block — only two states**, each with a small checkbox icon:

| State | Trigger | Appearance |
|---|---|---|
| In stock | Computed lead time is zero | "In Stock" |
| Lead time | Computed lead time above zero | A lead-time message (below) |

There is no "out of stock" here — those products route to a different page.

**Lead-time wording** (computed from the product's `leadTimeDuration`, the longest across selected options, plus days up to `dateDueBackInStock` if future; 5 days = 1 week). So **the stock line can change when the customer picks an option.**
- 1 day → "1 Working Day Lead Time"
- 2–4 or any non-week multiple → "{n} Working Days Lead Time"
- exactly 5 → "1 Week Lead Time"; multiples of 5 → "{n} Weeks Lead Time"
- 0 / blank → "Available with lead time" (this fallback appears only on the badges overlay, block D, not on this line — but design a chip for it)

**Port note.** The price here is the live, calculated per-unit price once the configurator has loaded; before that it is the raw stored price. See **Loading / first paint** in the cheat sheet.

### C. In-stock guarantee badge

**What it is.** A round "In Stock Guaranteed" reassurance seal; tapping it opens the guarantee's terms PDF. Recreate the wording and seal natively rather than reusing the pixel asset.

**When shown.** Only when **all three** hold: computed lead time is zero, no selected option carries `excludeFromStockGuaranteed`, and `inStockGuaranteed` is on. Because it depends on selected options and computed lead time, **it can appear and disappear live** as the customer configures, and it cannot appear on first load — it needs the configurator.

**States.** Present, or absent (no reserved space).

### D. Image gallery

**What it is.** The main product imagery: a large image with a thumbnail strip, tap-to-zoom, and video support.

**When shown.** Always — it always occupies its space and is never truly empty. On the live site single-image products were common, so **the gallery must look right with exactly one image** (the normal case, not an edge case).

**States.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | One or more images | Large hero image + thumbnail strip |
| No image | Image set empty/absent | A single "no image" placeholder tile |
| Broken image | An image fails to load | Swaps to the same placeholder |
| Video entry | Entry type is `video` | A video embed instead of a photo; its thumbnail shows a play icon |
| Pre-initialised | Briefly, before the carousel controls are ready | Only the first slide; arrows and zoom button not yet present |

Images are sorted by `sortOrder` with the hero moved to the front. Each image can carry a `caption` shown beneath it (hidden when blank).

**Interactions (as on the current web page).**
- Tap a thumbnail to change the main image.
- Tap the zoom button for a full-screen zoomable view (videos excluded from it).
- **Variant-linked images:** selecting an option jumps the gallery to the image tagged with that option. This is how "the picture changes when you pick a colour" works.

**Mobile touch model — decide this deliberately; the web model does not carry over.** The web design is thumbnail-strip + arrows + zoom-button, built for a mouse. Choose for the phone:
- **Changing image:** swipe left/right (expected on a phone) vs a tappable thumbnail strip. If swipe is primary, decide whether to keep thumbnails or replace them with page dots.
- **Position indicator:** a row of page dots is the usual phone equivalent of the strip.
- **Zoom:** pinch-to-zoom in place (expected) vs a separate tap-to-open full-screen view. Pick one.
- **Variant jump vs swipe position:** when a selection jumps the gallery to a tagged image, the gallery should move to and settle on that image (and its dot) so a later swipe continues from there.

**Port note.** Pick a fixed small thumbnail count that fits a phone (~4, or drop the strip for page dots); ignore the web's desktop breakpoint (6 thumbnails above 800px). The full-screen zoom internals were not read in full.

#### D-badges. Badges overlay (on the first image only)

**What it is.** A row of small badges laid over the first gallery slide only. Absent until the configurator has loaded.

**Content — fixed order, and they stack.** First, exactly one stock badge; then, independently and in this order, each flag adds a badge when on: `bestSeller` → "Best Seller", `isSale` → "Sale", `isBuyMorePayLess` → "Buy More Pay Less", `isNewProduct` → "New". All four can show at once — **design for all badges on.**

**The stock badge here is NOT driven identically to block B — design three chip variants:**
- **"In Stock"** — no lead time and the `leadTime` flag is off.
- **A lead-time message** (same wording ladder as block B) — when a computed duration is above zero.
- **"Available with lead time"** — when the `leadTime` flag is on but no duration number is set.

This badge fires when **either** the boolean `leadTime` flag is on **or** a computed duration exists — so a product can legitimately show "Available with lead time" as a badge while block B's line still reads "In Stock". Design that combination rather than assuming the two always agree.

### E. Multibuy discounts

**What it is.** A row of tappable buttons offering bulk-discount prices; tapping one sets the quantity to that threshold.

**When shown.** Only when `priceTiers` is non-empty. Present on some products regardless of complexity — on the live site the cheapest option-less product (a bag of cement) had one, while a £19,950 configurable gate did not.

**Content and states.** Header **"Multibuy Discounts Available"**. A synthetic "BUY 1+" button at the current price is added in front of the stored tiers, so N stored tiers give N+1 buttons. Each button reads "BUY {n}+" over "£{price} ea". **Exactly one is selected at a time** — the highest threshold at or below the current quantity. Thresholds vary widely (live: 16/50/151, and 31/281), so treat it as a variable-length list.

**On a phone a long row of buttons will not fit** — decide the mobile behaviour: let the buttons scroll sideways in one row, or wrap them onto multiple rows.

**Interaction.** Tapping a button sets the quantity, which updates the price everywhere on the page.

### F. Hire date picker

**What it is.** A date-range picker for hire products, with a flexible-hire toggle.

**When shown.** Only on hire products.

**Content and states.** Titled "Choose dates:". A start/end range, an "is flexible" toggle, a derived hire duration, and a minimum hire term; certain dates are blocked.

| State | Trigger | Appearance |
|---|---|---|
| No dates chosen | Start or end missing | **Blocks the add-to-basket buttons** with the error "Please select a date range!" |
| Valid range | Both dates chosen and term met | Shows the duration |

The live hire product also carried a "Need a flexible hire contract?" toggle and a refundable-deposit notice here.

### G. Quantity / Meters / Area inputs

**What it is.** The number inputs that set how much the customer is buying.

**When shown.** The Quantity field is always present; the other two are conditional.

| Field | Shown when | Behaviour |
|---|---|---|
| Quantity | Always | Snaps up to `minQuantity` (or 1) on blur; can show an error |
| Meters | `showMeters` on | Snaps to `bayWidth` on blur |
| Area Coverage (m²) | `showAreaCoverage` on | Snaps to `areaCoverage` on blur |

**States of the Quantity field.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Number with − / + steppers |
| Error | A selected option's minimum quantity exceeds the current quantity | Red border + a message below: "Minimum order quantity for '{option}' is {n}." |

**Port notes.**
- **No upper limit** on the steppers. `orderLimitQuantity` exists in the data and is sent to the cart but never enforced or shown. If the app should cap quantity, design it from scratch.
- The fields stack vertically on mobile.
- **Mobile keyboard:** these accept typed numbers, so tapping one raises the number keyboard, which can cover the field and the sticky buy bar. Specify keyboard-avoidance — scroll the tapped field into view above the keyboard, and use a numeric keypad.

#### G-note. Area-coverage advisory note

A small grey paragraph about irregular shapes and the customer's responsibility to confirm quantities. Shown only when `showAreaCoverage` is on — paired with the area input above.

### H. Variants / configurator

**What it is.** The heart of a configurable product: the option groups the customer answers to build their product (e.g. **Post Type**, **Pale Type**, **Do you require an end post?**).

**When shown.** Only when the product has variant groups. Absent on first load; appears once the configurator has loaded.

**Structure.** One sub-block per variant group, each with a heading (`displayedName`), an optional subtitle, and options rendered as one of the six control types (see the data model). A group renders nothing if it has no visible options.

**Per-option states.** Each option can be in one of four states. Because there are six control types, "selected" and "error" look different in each — give each control a concrete appearance rather than a generic "highlighted":

| State | Trigger | Appearance |
|---|---|---|
| Selected | The option is chosen | Radio/checkbox: filled control + check, label emphasised. Swatch: coloured square with a ring + check. Switch: toggle on. Dropdown: chosen row shown in the closed control. Product row: card with a selected border/tint. |
| Unselected | Not chosen | The plain resting form of the same control |
| Removed (hidden) | An incompatible option is currently selected — **only in PRODUCT-type groups** | The option is taken out of the list entirely |
| Error / restricted | The user has selected two options that conflict (mainly in multi-select checkbox groups) | The conflicting option is flagged and add-to-basket is blocked. Checkbox rows: inline "(Not available with {option} Selected)". Product rows: behind a **"View Restrictions"** label the user taps to reveal (hover on web — make it a tap target on a phone) |

**Hidden vs restricted are different mechanisms.** Incompatibility **removes** an option only in PRODUCT-type groups. The **error/restricted** state is separate — it fires when the customer actively selects two clashing options (possible when a group allows multiple picks) and shows a message rather than removing anything. There is no greyed-but-present disabled option today; if the redesign prefers greying-out to removal, that is a new choice.

**The option price label.** To the right of each option: "+ £X (each)" for a positive modifier (or "(each/week)" on hire); the word "Standard" when the modifier is zero; or blank when `hideStandard` is set (radio/checkbox always render blank at zero). Live examples baked the delta and even lead-time into the label, e.g. "Bolt Down +£15.75 (each)", "Blue RAL 5010 (1 Week Lead Time)".

**Collapse behaviour.** Only swatch (and product-picker) groups truncate to the first 4 options behind a one-way "Show More Options" link. Other control types always show every option.

**Interactions.** Selecting an option can, live: change the running price, change the stock line and lead time, change the guarantee badge, jump the gallery to that option's image, and add or remove upsells.

**Port note.** The mix of controls on one product is real — the live gate had two dropdowns, a radio pair, and a colour-swatch set on one page. Design all six control types.

#### H-qty. Per-option quantity stepper

**What it is.** A quantity stepper attached to an individual option.

**When shown.** Only when the option is selected AND its group has `qtyInput` on AND the option's price modifier is not zero.

**States.** Label "Quantity" with − / number / +. Decrement floors at the minimum; increment is unbounded. If a recommended quantity is set, shows "Recommended Quantity: {n}", styled as a warning while the current value is below it.

### I. Installation

**What it is.** An optional paid installation service, configured through a modal (ground type, surcharges, price bands).

**When shown.** Only when the product has an `installationType` that passes validation.

**Three states.**

| State | Trigger | Appearance |
|---|---|---|
| Absent | No installation type, or invalid | Nothing renders |
| Skeleton / disabled | Installation data exists but the configurator has not loaded | Title "Installation", an unchecked box, £0, the hard-coded word "METERS", a disabled button |
| Normal | Loaded | Title "Installation", a checkbox, the current price, the lowest available price + unit, and a button opening the configuration modal. While the calculation is incomplete, the subtitle "Choose and fill details" shows |

**The unit is data-driven, not always "METERS".** The unit next to the lowest price comes from the configuration and is either **METERS** or **QUANTITY**. Only the disabled skeleton hard-codes "METERS"; the loaded state renders whichever the data carries, so it can read "QUANTITY".

**Interaction.** Ticking the box before the calculation is valid **force-opens the modal** rather than toggling; once valid it toggles directly. The modal covers ground-type selection (with a `groundTypeToolTip` — hover on web, so on a phone attach it to a tappable info icon or show it inline), surcharge options, and price bands with a minimum-charge floor.

**Port note.** The installation modal's internal states were not read in full.

### J. Frequently Bought Together (upsells)

**What it is.** A list of recommended add-on products the customer can add alongside the main product.

**When shown.** Only when `upsellProducts` is non-empty.

**States.**

| State | Trigger | Appearance |
|---|---|---|
| Expanded (default) | Always the starting state | Header "Frequently Bought Together" + a "Hide" toggle, then the list |
| Collapsed | User taps "Hide" | Header only |
| Card hidden | The upsell is out of stock, OR a selected option carries "hide this upsell" | That card disappears and its quantity is forced to zero |

Each card: product title, an optional lead-time chip, an image, a quantity stepper (floors at 0, unbounded up), a price with "ea (Ex VAT)", and an optional description (`shoppingDesc`) with a show-more toggle. A recommended-quantity line shows only when the computed recommendation is above zero and turns to a warning style when the current quantity is below it. **The recommendation recomputes live** as the parent quantity or the selected options change. Options can inject additional upsells into this list.

### K. Purchasable extras

**What it is.** Products bought outright alongside a hire, each its own cart line.

**When shown.** Only on hire products, and only when `purchasableExtras` is non-empty.

**Content.** Title "Purchasable Extras", then one row per extra with a quantity control. Only extras with a quantity above zero are added.

### L. Add-on service toggles (PostPro / BreezeGuard / TWD)

**What they are.** Up to three engineering-calculation add-on services, each shown as a checkbox with its own add-to-basket flow and a file-upload / configuration modal. These are niche business services — **confirm with the team whether they are in scope for the mobile app before building them.**

**When shown.** Each is independent:
- **PostPro** — only when `showPostPro` is on. One checkbox at a hard-coded price (£1,368.75 in the web source — **not from data**), its own "Add to basket" button opening a file-upload modal.
- **BreezeGuard** — only when `showBreezeGuard` is on. Checkbox "Do you require basic wind load calculation?", hard-coded "£99.00 per design".
- **Temporary Works Design (TWD)** — only when `showTempWorksDesign` is on. Checkbox with hard-coded "£749.99 per design" plus "1 TWD is required per product".

**Behaviour.** BreezeGuard and TWD are **mutually exclusive** — ticking one unticks the other; they share one add-to-basket button that opens the matching modal, and if neither is ticked it does nothing.

**Port note.** All three prices are hard-coded in the web source, not from data — confirm before baking in.

### M. Add To Basket box (in-page)

**What it is.** The primary buy control with the running total.

**When shown.** Always.

**States.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Big total + the unit word (capitals), a delivery note, an "£X incl VAT" line, and a full-width "Add To Basket" button |
| Loading | While an add request is in flight — and briefly on first load | The button is disabled (no spinner, it just goes dead) |
| Blocked with error | A validation error exists AND the user has interacted | The first error message shows below the button |

The delivery note here **always** reads "Exact delivery cost added during basket checkout" — even on hire (the refundable-deposit wording lives only on the sticky bar).

**Port notes.** A commented-out "Add To Quote List" button — **do not port**. Errors never show on a fresh, untouched page.

### N. Detail sections — Specifications & Downloads, What's Included, Description

**What it is.** The long-form product information. **This block's mobile behaviour differs most from desktop, and it is the most important layout fact here.**

On desktop these are a tab strip (one visible at a time). **On mobile the tab strip is gone — the sections are stacked, all expanded at once, and in a different order.** The mobile order is:

1. Specifications & Downloads
2. What's Included
3. Product Description

Design the mobile stack, not tabs. Each section self-hides when its data is empty, so the stack can be short.

| Section | Shown when | Content |
|---|---|---|
| Specifications & Downloads | `moreInformation` OR `downloads` non-empty | A name/value spec table AND a list of download links, sharing one section. Either half can be empty while the other fills it. |
| What's Included | `components` non-empty | A Name/Quantity table |
| Product Description | Always present; body empty if `html` blank | Rich formatted text (headings, lists, embedded images), unbounded length, no "show more" — needs a real rich-text renderer |

**Specifications table.** Name/value pairs from `moreInformation`. Live tables ranged 3–11 rows. Values often carry bracketed qualifiers in the text, e.g. "2.0m [Installed]".

**Downloads list.** A flat list of tappable links (the file `link`) showing the document `name` with a download icon. **No grouping, no headers, no size/type.** A row's link can be blank if no file was attached. The **empty downloads state is real** — a live product had zero documents but the section still rendered (fed by its specs). Names are free text with inconsistent casing and can be long — they must wrap.

**What's Included table.** Name/Quantity rows from `components`. **Each quantity is multiplied by the current order quantity** — so the parts list changes live as the customer changes how many they buy.

### O. Sticky Add To Basket bar

**What it is.** A buy bar pinned to the bottom of the screen at all times, duplicating the in-page box (block M).

**When shown.** Always. It floats over the page content.

**States.**

| State | Trigger | Appearance |
|---|---|---|
| Normal | — | Total + the label "TOTAL", an "£X incl VAT" line, a delivery note, and an "Add to Basket" button |
| Error | A validation error exists AND the user has interacted | A red card with the first error message appears above the bar |
| Disabled | Error (after interaction) OR loading | Button disabled |

The delivery note here **switches on hire**: "Refundable Hire Deposit and Delivery calculated in the cart" on hire, "Exact delivery cost added during basket checkout" otherwise. On mobile the delivery note is hidden and the price shrinks and centres.

**Port notes.**
- **Reserve space for it.** The last blocks on the page need enough bottom padding that the bar never covers them, and the bar must sit above the device's bottom safe-area inset (home indicator) so its button stays tappable.
- **It duplicates the in-page box (block M)** with different labels ("EACH" in-page vs "TOTAL" on the bar), and both appear at once. On a phone, decide deliberately which one the app keeps.

### P. Cart confirmation modal

**What it is.** An overlay shown after a **successful** add to basket.

**When shown.** Only after a successful add. A failed add does **not** use this modal — it shows a transient red toast, "Adding of the product failed!", for five seconds.

**Content.** Title "Added to Basket", a green check, and three actions: "Go to basket", "Proceed to checkout", and "Continue Shopping" (closes it).

---

## States cheat sheet (scoped)

Keep this open while designing. Each global state below changes several blocks at once.

### Loading / first paint

The page shows content immediately from baked-in data, then the interactive configurator loads a moment later and things recalculate. Expect a visible "settle":

- Price and quantity first show raw stored values, then switch to the calculated values (including any auto-selected option's price change).
- Blocks that need the configurator — **Variants (H), per-option steppers (H-qty), Installation (I, shows its disabled skeleton), Frequently Bought Together (J), the badges overlay (D), the in-stock guarantee badge (C)** — are absent or skeletal on first load, then appear.
- Both Add To Basket buttons are briefly disabled on load.

There is no skeleton UI in the current web app. **The port should decide deliberately** whether to show settled values directly (it can, since the data is local) or design loading states.

### In stock vs lead time (there is no "out of stock" on this page)

Every product that reaches this page is either "In Stock" or shows a lead-time message. This affects:

- **Block B** (stock line): "In Stock" or a lead-time message — keys off the computed duration only.
- **Block D** (badges overlay): a similar chip, but **not driven identically** — it fires on the boolean `leadTime` flag OR a computed duration, and has a third "Available with lead time" wording. So the badge can read a lead time while block B still reads "In Stock".
- **Block C** (guarantee badge): only shows when lead time is zero.

Because lead time is computed from selected options and a due-back date, **selecting an option can flip a product from "In Stock" to a lead time**, changing all three live.

### No price / Free

- **No price** (`price` blank): the price block degrades toward Free/zero.
- **Free** (`price` is 0): shows the word "Free", hides the incl-VAT line.
- **`ignoreBasePrice` on**: the entire price + stock line (block B) is hidden; price is built purely from picked options.

### Old-price / "WAS"

Driven only by `offerPrice`. When set, block B shows a "WAS £X" beside the current price. Remember it is the *old, higher* number. None of the four live pages read actually showed this state — confirm the exact appearance before finalising.

### No image

A normal, common state. The gallery (block D) shows a hard-coded placeholder tile; it is never truly empty. The badges overlay still applies to that placeholder.

### Has variants (configurable)

When the product has variant groups (block H present):

- Blocks B (price), B (stock line), C (guarantee badge), and D (gallery image) all become **live** and can change as options are picked.
- Frequently Bought Together (J) can gain or lose cards as options are picked.
- Validation errors become possible (below), affecting both Add To Basket controls (M and O).
- Most groups arrive with a selection already made; only checkbox groups start empty.

### Hire product

- Block F (date picker) appears; block K (purchasable extras) can appear.
- Price shows "per {unit} / per week"; the sticky bar's delivery note changes to the refundable-deposit wording.
- A "Purchase Me!" cross-link may appear in the header.

### Validation / cannot add to basket

Three possible errors, of which only the first is ever shown, and only after the user has interacted:

1. "Please check your selected options!" — an incompatible option combination is selected.
2. "Minimum order quantity is {n} for the selected option." — quantity below an option's minimum. (The quantity field shows a differently-worded version of the same problem — standardise this in the port.)
3. "Please select a date range!" — a hire product with no dates chosen.

When present (after interaction), the error disables both Add To Basket controls and shows above the sticky bar and below the in-page button.

### All-badges-on

Because badge flags are independent and non-exclusive, design for the worst case: the stock chip plus "Best Seller", "Sale", "Buy More Pay Less" and "New" all stacked on the first gallery image at once.

---

## What this scoped version leaves out

These blocks exist on the web product page but are **out of scope here** (marketing, navigation, or third-party furniture). See `PRODUCT_PAGE_STRUCTURE.md` for each in full:

- **Breadcrumb trail** — category path navigation.
- **Flexible-finance line** — the Iwoca "pay in 3" message.
- **Image disclaimer** — a static one-line legal note under the gallery.
- **Jump-to-section buttons** — shortcuts to the detail sections (on mobile these just scroll).
- **Logo / accreditation strip** — brand/accreditation logos (`logoCollection`).
- **Promo banners** — Trade Installer, TWD banner, MUGA Play (tag- or SKU-driven flat artwork images).
- **Reviews widget** — a third-party Feefo script, no First Fence data behind it; reviews are keyed by category, not product.
- **Phone-help line**, **Calculator link button**, **Contact Us banner** — static help/contact prompts.
- **Related products carousel** ("You May Also Like") — fetched live; poorly-behaved loading/empty state on the web.
- **Help centre articles carousel** and **FAQ accordion**.

Also out of scope: the three **non-buyable page types** (out-of-stock, price-on-application, group/kit), covered at the end of the full document.

---

## Appendix: for developers

Files behind the in-scope blocks, so the doc can be traced to source. Paths are relative to `/home/osemeniuk/Documents/Work/Projects/first-fence/`.

**Data model and backend (`cdn-graphql-v2/`):**
- Product type and enums: `src/schema/product.js`, `src/schema/index.js`
- Variants: `src/schema/product-variant.js`; images: `src/schema/product-images.js`, `src/schema/image.js`
- Installation: `src/schema/installation-type.js`, `src/schema/surcharge.js`
- Mongoose models: `src/models/product.js`, `src/models/product-variants.js`, `src/models/product-images.js`, `src/models/image.js`
- Resolvers (relation hydration, image URL rewrite/filter, defaults): `src/resolvers/product.js`, `src/resolvers/product-variant.js`, `src/resolvers/product-files.js`
- Create/upload defaults: `src/resolvers/product.js`; `src/utils/constants.js`
- Note: no pricing/stock logic exists server-side — `src/services/` holds only `s3.js` and `webhook.js`

**Frontend (`gatsby-website/`):**
- Template and block order: `src/templates/product/product.js`; layout CSS: `src/templates/product/product.module.css`
- Interactive model (defaults, auto-selection, field renames, price/qty computation): `src/models/Product.js`, `src/models/ProductVariant.js`
- Header / Buy-Hire switch: `src/components/product-header/product-header.js`, `src/features/hire-product/buy-switch/buy-switch.js`
- Price / VAT: `src/components/price/price.jsx`, `src/components/price/unit/unit.js`, `src/utils/general.js`
- Stock / lead time / guarantee: `src/components/stock-message/stock-message.js`, `src/utils/product/lead-time.js`, `src/components/stock-guarantee-message/stock-guarantee-message.jsx`, `src/utils/product/stock-guaranteed.js`
- Gallery + badges: `src/components/image-gallery/image-gallery.js`, `.../fullscreen-gallery/fullscreen-gallery.jsx`, `.../badges/badges.jsx`, `src/utils/product/images.js`, `src/utils/gallery.js`
- Multibuy tiers: `src/components/price-tiers/price-tiers.jsx`, `.../price-button/price-button.jsx`
- Hire date picker: `src/components/calendar-date-picker/`, `src/constants/product.js`
- Quantity / meters / area: `src/components/product-inputs/product-inputs.jsx`, `src/components/number-field/number-field.js`
- Variants: `src/features/product/variants/` (variants.js and the per-type renderers under `swatch/`, `select-box/`, `switch/`, `radio/`, `checkbox/`, `product/`, `quantity-box/`)
- Installation: `src/features/product/installation/Installation.js` and `.../installation-modal/`
- Upsells / extras: `src/features/product/upsell-products/`, `src/features/hire-product/purchasable-products/`
- PostPro: `src/features/post-pro/post-pro.js`; BreezeGuard / TWD: `src/features/breezeguard/`
- Add to basket (in-page and sticky): `src/features/product/add-to-basket-button/add-to-basket-button.js`, `src/components/add-to-basket-banner/add-to-basket-banner.js`
- Validation: `src/utils/product/validation.js`, `src/constants/error-codes.js`
- Detail sections (tabs + mobile stack): `src/features/product/product-details-tabs/product-details-tabs.jsx`, `src/templates/product/product.js` (mobile sections), and content components `src/components/product-description/`, `src/components/more-information/`, `src/components/specifications/`, `src/components/downloads/`, `src/components/product-components/`
- Cart modal: `src/components/cart-modal/cart-modal.js`
