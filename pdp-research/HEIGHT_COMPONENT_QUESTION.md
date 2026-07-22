# Is "Height" the same component as Quantity / Meters / Area Coverage?

Answering a question raised against the generated mockup at `first-fence-pdp-components.vercel.app`.
No existing file has been changed.

---

## Short answer

**No — they are two different blocks, and the mockup has them in the wrong order.**

| | The "Height" card grid | Quantity / Meters / Area Coverage |
|---|---|---|
| What it is | **One variant group** — same block as *Post Type*, *Pale Type*, *Do you require an end post?* | A **separate block** of plain number inputs |
| Doc reference | Block **H** (scoped) / Variants (full) | Block **G** (scoped) / Product inputs (full) |
| Component | `features/product/variants/` | `components/product-inputs/` |
| Driven by | `productVariants[].displayedName` + `variantType` + `options[]` | Product booleans `showMeters` / `showAreaCoverage` |
| Position on the live page | **After** the inputs | **Before** the variant groups |
| How many can appear | Any number — one per variant group | Exactly one, with 1–3 fields inside |

"Height" is not a component type. It is **a label an admin typed into `displayedName`**. The *shape* on
screen comes from `variantType`, not from the word "Height".

---

## Why it looks like its own thing

There are **140 variant groups** in the database whose name mentions height, spread across **21 different
spellings** of the same idea:

> Height · Height (trailing space) · Gate Height · Post Height · Panel Height · System Height ·
> Installed Height · Required Height · Select Height · Select a Height · Select System Height ·
> Select Ladder Height · Choose Height · Choose your Height · Choose your height ·
> Choose your Panel Height · Choose your fence height · Pick your Fence Panel Height ·
> Post Height - Railing Test · Use with Railing Height

…and they are built from **four different control types**:

| `variantType` | Height groups using it | Renders as |
|---|---|---|
| `SWITCH` | 105 | Compact card grid — **this is what the mockup drew** |
| `SELECT` | 21 | Wide card grid in a bordered box (**not** a dropdown — see section 5) |
| `PRODUCT` | 13 | Product-picker rows with images |
| `RADIO` | 1 | Stacked radio rows |

So "the height component" isn't one thing. The same shopper-facing question renders in four different shapes
depending on which product they're looking at. That is why it reads as a distinct block — but the mechanism
is identical to Post Type and Pale Type.

For scale, across the whole catalogue: SWATCH 2,001 · PRODUCT 1,022 · SELECT 638 · SWITCH 512 · RADIO 178 ·
CHECKBOX 149.

---

## What the live site actually renders

**`https://firstfence.co.uk/358-mesh-sheets`** — "358 Mesh Sheets - Panels Only". Its Height group is a
`SWITCH`, which is exactly the mockup's shape. Extracted from the live page HTML:

```
Quantity  [ number input ]            ← block G, FIRST

Height                                ← block H, group 1 (SWITCH)
  1834mm      Standard
  2010mm      + £11.95 ea
  2405mm      + £40.06 ea
  3001mm      + £78.46 ea

Finish                                ← block H, group 2 (SWATCH)
  Prices vary, view goods total to see price per unit
  …

Add Universal Mesh Clips?             ← block H, group 3 (PRODUCT)
Add M8 x 40mm Mesh Fencing Bolts?     ← block H, group 4 (PRODUCT)
Add Torque Bit?                       ← block H, group 5 (PRODUCT)
```

That product has `showMeters: false` and `showAreaCoverage: false`, so block G is just the single Quantity
field — and it sits **above** Height, not below it.

---

## Three things to fix in the mockup

### 1. The order is inverted

In the product template, the inputs render at line 460 and the variant groups at line 485. Confirmed in the
delivered HTML: the `placeholder="Quantity"` input markup appears **before** the "Height" title.

The mockup shows Height on top and Quantity underneath. On the live site it is the other way round.

### 2. The zero-price option should say "Standard", not a price

Every option's price label is produced the same way: add the option's price modifier to any additional
price, then format it. **When that total is zero the label becomes the word "Standard"** — unless the option
has `hideStandard` set, in which case it renders blank.

The mockup gives all four cards a price. On the real product above, the first option (`1834mm`, no modifier)
reads **"Standard"**. That is the most common shape for a size picker — the base size is included, the
larger ones cost extra — so the mockup is missing the state that will appear on most of these groups.

Radio and checkbox groups always render blank at zero rather than "Standard"; switch, select and swatch
render "Standard".

### 3. Minor: the live label has spaces around the plus

Live output is `" + £11.95 ea"`, not `"+£11.95 ea"`. On a hire product the suffix becomes `" ea/week"`; in
`SELECT` and `RADIO` groups it is `" (each) "` / `" (each/week) "` instead of `" ea"`. Worth normalising in
the app rather than copying four spellings.

---

## The duplicate "1.8m" is accidentally correct

The mockup shows **two cards both labelled "1.8m"** at different prices (+£5.03 and +£12.21). That looks
like a rendering mistake — but it is a real, live data pattern.

**13 of the 140 height groups contain duplicate option labels.** A published example,
`/48mm-black-yellow-hooped-barrier`, has a Height group with nine options:

| Label | Price modifier | SAP SKU |
|---|---|---|
| 500 mm | Standard | BOLL-HOOP-0170 |
| 500 mm | + £21.29 | BOLL-HOOP-0180 |
| 500 mm | + £27.21 | BOLL-HOOP-0190 |
| 1000 mm | + £27.21 | BOLL-HOOP-0200 |
| 1000 mm | + £62.94 | BOLL-HOOP-0220 |
| 1000 mm | + £92.74 | BOLL-HOOP-0240 |
| 1000 mm | + £27.21 | BOLL-HOOP-0210 |
| 1000 mm | + £62.94 | BOLL-HOOP-0230 |
| 1000 mm | + £92.74 | BOLL-HOOP-0250 |

Six options labelled "1000 mm", of which **three pairs are identical in both label and price** —
distinguishable only by the SAP SKU printed underneath.

The web relies entirely on that SKU line to tell them apart, which is barely usable on a desktop and will be
worse on a phone. **The app needs a deliberate answer for this**: show the SKU, merge duplicates, or push
back on the data. It cannot be assumed away — the mockup stumbled into a genuine problem.

---

## Correction: `SELECT` is not a dropdown

Worth flagging because it affects this question directly and appears in the scoped doc's block H states
table, which describes SELECT as *"Dropdown: chosen row shown in the closed control."* **That is wrong.**

`SELECT` renders a **grid of radio-backed cards** — `grid-template-columns: repeat(auto-fill, minmax(400px, 1fr))`
— inside a bordered, 16px-radius container with a 24px bold title. There is no `<select>` element and
nothing collapses. It is closer to the mockup's Height card grid than to a dropdown.

The real difference between the two card types:

| | `SWITCH` | `SELECT` |
|---|---|---|
| Layout | flex-wrap, cards sized to content | grid, min 400px columns |
| Container | no border, 20px bottom padding | 1px border, 16px radius, 24px padding |
| Title | 18px / weight 600 | 24px / weight 700 |
| Price suffix | `" ea"` | `" (each) "` |
| Lead time / stock message | hover **tooltip** behind a "Lead Time" / "Stock Message" chip | printed inline under the option name |
| SAP SKU | shown | shown |

The hover tooltip in `SWITCH` is the one that matters for mobile: on a phone there is no hover, so the lead
time and stock message attached to an option are **unreachable** unless you redesign that as a tap or print
it inline the way `SELECT` does.

---

## Card appearance (SWITCH), for reference

| | |
|---|---|
| Resting | background `rgb(246,246,246)`, 1px border `rgb(200,200,200)`, 10px padding, centred column |
| Selected **and** hover | background `rgb(237,237,237)`, 1px border `#575756` — **identical**, so on the web hover is indistinguishable from selected |
| Error / restricted | 3px red border, red text |
| Option name | `#212529`, weight 500, centred |
| Price line | `#585858`, 14px |
| Lead-time chip | orange `#d47300` |
| Radio input | `display: none` — the whole card is the control |

**Selected and hover being the same style is a real problem to fix, not copy.** On touch there is no hover,
so it happens to work — but it means the web has no visual distinction to borrow from. The app needs to
design "selected" from scratch.

---

## How to tell the two blocks apart in the data

**It's a variant group (block H)** if it comes from `productVariants[]`: it has a `displayedName`, a
`variantType`, an optional `subtitle`, and an `options[]` array where each option carries `name`,
`priceModifier`, `additionalPrice`, `sapSkuIdentifier`, `leadTimeDuration`, `stockMessage`, `hideStandard`.

**It's the inputs block (block G)** if it comes from product-level fields: Quantity is always present;
`showMeters` adds a "Meters" field; `showAreaCoverage` adds an "Area Coverage (m2)" field. These are plain
number inputs with `min` clamping on blur — no options, no prices, no images.

A quick test on any product: if the thing has **per-option prices**, it is a variant group. Block G never
shows a price.

---

## Recommendation for the app

Build **one** variant-group component that takes a group and renders it according to `variantType`. Do not
build a "Height picker" — there is no such thing in the data, and the next product will call it
"Choose your Panel Height" and render it as a product picker with images.

The pieces that component must handle, all confirmed live:

- Group title (`displayedName`, falling back to `variantName`) and an optional **HTML** subtitle — the
  subtitle is injected as raw HTML and can contain links.
- Six control types: `SWITCH`, `SELECT`, `SWATCH`, `PRODUCT`, `RADIO`, `CHECKBOX`. Any other value renders
  nothing.
- Per-option price label with three forms: `+ £X ea`, `Standard`, or blank.
- Duplicate option labels (section 4) and the SAP SKU line that disambiguates them.
- Per-option lead time and stock message — currently hover-only in `SWITCH`.
- The error/restricted state and, in `PRODUCT` groups only, options being removed entirely.

---

## Sources

**Code** (`gatsby-website`):
- `src/features/product/variants/variants.js` — the six-way dispatch on `variantType`
- `src/features/product/variants/switch/switch.js` + `switch/option.js` — the card grid in the mockup
- `src/features/product/variants/switch/option.module.css` — card styling in section 6
- `src/features/product/variants/select-box/select-box.js` + `option.js` — the not-a-dropdown correction
- `src/components/product-inputs/product-inputs.jsx` — Quantity / Meters / Area Coverage
- `src/templates/product/product.js` — `ProductInputs` at line 460, `Variants` at line 485 (the ordering)
- `src/utils/general.js:16` — `getPriceString`, which produces "Standard" at zero

**Live**:
- `https://firstfence.co.uk/358-mesh-sheets` — SWITCH Height group, verified in delivered HTML
- `https://firstfence.co.uk/48mm-black-yellow-hooped-barrier` — the duplicate-label Height group

**Data** (Mongo `cdn`, `productvariants`): 140 groups named after height across 21 spellings and 4 control
types; 13 of them contain duplicate option labels.

**Existing docs**: block **H** of `PRODUCT_PAGE_STRUCTURE_SCOPED.md` (and the Variants block of
`PRODUCT_PAGE_STRUCTURE.md`) already describe this correctly, apart from the SELECT-is-a-dropdown error
noted in section 5.
