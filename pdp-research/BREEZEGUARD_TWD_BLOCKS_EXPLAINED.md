# BreezeGuard and Temporary Works Design — what they are, where they live, what the upload is for

Companion to `POSTPRO_BLOCK_EXPLAINED.md`, checked the same way: source, production database, live HTML.
Follow-up to block 19/20 of `PRODUCT_PAGE_STRUCTURE.md` and block L of `PRODUCT_PAGE_STRUCTURE_SCOPED.md`.
No existing file has been changed.

**Headline:** unlike PostPro, these two are **not separate blocks**. They are two checkboxes inside **one
card sharing one "Add to basket" button**. And "TWD" refers to **three different things** on the product
page, driven by two different data fields — which is the main thing to get straight before designing
anything.

---

## 1. The three things called "TWD"

| # | Thing | Driven by | Where on the page |
|---|---|---|---|
| 1 | **TWD banner** — a clickable image linking to `/twd` | a **tag** named `#twd` | Higher up, near the price-tiers area |
| 2 | **"Do you require a Temporary Works Design?"** checkbox | the **boolean field** `showTempWorksDesign` | Lower down, in the shared card above Add To Basket |
| 3 | **BreezeGuard** — sold as *"BreezeGuard Basic Temporary Works Design"* in the database | the boolean field `showBreezeGuard` | Same shared card, above the TWD checkbox |

Numbers 1 and 2 are independent. A tag drives one, a field drives the other, **and they do not have to
agree** — 55 live products carry the `#twd` tag while 69 have the checkbox field on. One page can show
both, and both test pages I checked did.

Number 3 is confusing on purpose-of-naming grounds: BreezeGuard's underlying product is literally titled
"BreezeGuard Basic Temporary Works Design". It is the cheap version of the same idea.

---

## 2. One card, two checkboxes, one button

`BreezeGuard` and `Temporary Works Design` are rendered by a single component. The card disappears only if
**both** flags are off. Each checkbox appears independently — but the **"Add to basket" button is shared**
and sits at the bottom of the card, outside both sections.

Live on `https://firstfence.co.uk/standard-temporary-fencing-panel`, exactly as coded:

```
BreezeGuard™                                                       ← h3
Panel, Stabilising and Wind Load Calculations                      ← italic subtitle
☐  Do you require basic wind load calculation?
   BreezeGuard suitability report costs £99.00 per design

Temporary Works Design                                             ← h3
Professional Engineering Calculations and Site-Specific Solutions  ← italic subtitle
☐  Do you require a Temporary Works Design?
   Engineer issued TWD and advice for your specific site for only
   £749.99 per design
   PLEASE NOTE: 1 TWD is required per product.

                                        [ Add to basket ]          ← ONE shared button
```

The two checkboxes are **mutually exclusive** — ticking one unticks the other. So the card is really a
two-option radio group wearing checkboxes.

### How often each combination actually occurs

Published, visible products that get a real page:

| Combination | Live count |
|---|---|
| **TWD checkbox only** | **60** |
| Both checkboxes | 9 |
| BreezeGuard checkbox only | **1** |
| Total cards shown | **70** |

(Totals including draft/test/archived: BreezeGuard 11, TWD 94.)

**So the two-checkbox card is the exception, not the rule.** 60 of 70 pages show a single TWD checkbox and
a button — and, as with PostPro, a one-option checkbox group carries no information. Exactly one product in
the entire catalogue shows BreezeGuard alone.

Verified live: `/standard-temporary-fencing-panel` shows both; `/ex-hire-hoarding-50-package-deal` shows TWD
only. Products are temporary fencing panels, hoarding kits, hoarding gates and their accessories.

---

## 3. What each service actually is

**Temporary Works Design (£749.99)** — a signed engineer's design for a specific site: what fencing or
hoarding to use, on what base, with what ancillaries, for how long, at that location. Construction sites
generally require a competent-person temporary works design before erecting hoarding. Note the page's own
warning: **"1 TWD is required per product"** — the customer may legitimately need several.

**BreezeGuard (£99.00)** — the cut-down version: *only* wind-load and stability calculations for a chosen
panel + stabiliser combination at a given wind speed. No site visit, no location, no documents. It answers
one question: *will this panel stay upright on this base in this wind?*

Both carry the identical **third-party disclaimer** used by PostPro: not manufactured or provided by First
Fence, no guarantee of accuracy, recourse is against the third party. These are referral products.

---

## 4. Where the upload is — and where it isn't

**BreezeGuard has no upload at all.** No file input anywhere in its modal. It is a pure questionnaire.

**TWD does have an upload**, inside its modal, exactly like PostPro: a single "Additional files" picker
accepting **PNG, JPEG, PDF up to 10 MB each**, multiple files, with thumbnail previews and a rejected-file
list. It posts to `/upload/twd`.

Note the difference from PostPro, which has **three separate** upload categories (ground investigation, site
plan, layout plan) each behind a yes/no question. **TWD has one flat optional file set** and no
yes/no gating.

As with PostPro: neither the upload nor any file control exists on the product page. Ticking the box and
pressing "Add to basket" opens the modal; the modal contains the form.

---

## 5. The BreezeGuard modal, section by section

**1 — "What type of temporary fencing top are you using?"** Three image cards, pick one. Each is a specific
catalogued product, name and SKU baked into the source:

| Option | SKU |
|---|---|
| Standard Temporary Fencing Panel | WEB-00756 |
| Heavy Duty Round Top Temporary Fencing Panel | WEB-00784 |
| Round Top with Centre Bar Temporary Fencing Panel | WEB-00786 |

**2 — Stabiliser type.** Four image cards:

| Option | SKU |
|---|---|
| Feet For Temporary Fencing | TEMP-FEE-0001 |
| 390 Kentledge Big Foot Block | BARR-CON-0010 |
| 780 Kentledge Big Foot Block | BARR-CON-0011 |
| Durablock Ballast Block 50kg | TEMP-BAL-0003 |

**3 — Wind speed.** A number field, **1–42 m/s**, step 0.1, hard-clamped to the range. Below it: *"Don't
know your wind speed? Search for your annual average below"* and a dropdown of **21 UK regions** (Thames
Valley, Midlands, Highlands, Orkney, Shetland, Northern Ireland …) which auto-fills the annual average.
Data credited to BRE with a link to their wind-data PDF. An ⓘ toggle reveals help text.

**4 — Factor of Safety.** A dropdown from **1.0 (0%) to 2.0 (100%)** in 0.1 steps. A second ⓘ toggle
reveals a long explanation of what happens if you pick no safety factor.

**5 — Terms and conditions.** The third-party disclaimer, a link to the third party's full terms, and a
mandatory acceptance checkbox.

**6 — Captcha.** Commented out. No bot protection.

**Footer:** "Need help with BreezeGuard™? Speak to our team on 01283 512 111", then **Add To Basket** and
**Cancel**. After submission the entire body is replaced by "Product successfully added to basket" (or
"Sorry there was an error."), with **Continue Shopping** / **View Basket**.

---

## 6. The TWD modal, section by section

Much larger than either of the other two.

**1 — Location.** Address lookup → postcode resolves to coordinates → Google Map with a draggable marker,
plus what3words fields. (Same component pattern as PostPro.)

**2 — Duration.** How long the works will be on site.

**3 — Product type.** A dropdown: **Fencing / Steel Hoarding / Timber Hoarding / Gates / RapidShield**.

**4 — The catalogue picker.** This is the bulk of it: an image-card picker for the product, then its base,
then ancillaries — and for timber hoarding, additional pickers for **posts, rails, sheet infill and
kentledge**. All of it is **hard-coded in the page source**: 1,089 lines containing **53 hard-wired SKUs**.
Nothing here comes from the API.

**5 — Meterage or quantity**, depending on what was picked.

**6 — File upload.** The optional "Additional files" picker described in section 4.

**7 — Additional notes.** Free text, with a long placeholder asking for anything affecting usage or
installation.

**8 — Terms and conditions**, same disclaimer plus mandatory checkbox.

**9 — Captcha.** Also commented out — the state variable is still passed to the validator, but the
validator's check is commented out too.

On submit: `POST /carts` to add the product, then `POST /upload/twd` with the files. Same
basket-then-upload order as PostPro.

---

## 7. Prices: all three labels currently match the database

Each service is a hidden product fetched by hard-coded id when its modal opens:

| Service | Product | DB price | Label in page source | Match? |
|---|---|---|---|---|
| BreezeGuard | `65cde569f86b79d2ba8a6501` "BreezeGuard Basic Temporary Works Design" (`BRE-001`, hidden) | £99.00 | £99.00 | ✅ |
| TWD | `65eb1595f86b79d2ba716ab9` "Temporary Works Design" (hidden) | £749.99 | £749.99 | ✅ |
| PostPro | `66a0cc683cd5eb05192dddc6` (hidden) | £1,368.75 | £1,368.75 | ✅ |

Same conclusion as the PostPro doc: **the label is hard-coded, the charged price is not, and nothing keeps
them in sync.** All three agree today. That is luck, not design.

---

## 8. Bugs

Verified in source. Several are shared with PostPro; two are specific to these blocks.

### Shared with PostPro

**1 — Nothing ticked → the shared button silently does nothing.** No message, no disabled state. Worse
here, because the button belongs to a card containing two sections and gives no hint which one it acts on.

**2 — Labels not attached to their checkboxes.** `htmlFor="breezeguard"` vs `id="breezeGuardOption1"`, and
`htmlFor="twd"` vs `id="breezeGuardOption2"`. Tapping the text only works because the surrounding container
carries its own click handler.

**3 — Captcha disabled** on both modals.

### Specific to these blocks

**4 — The button reads the DOM, not React state.** The handler calls `document.getElementById(...)` on the
two checkbox nodes to decide which modal to open, while the visual tick state lives in React state. Two
sources of truth for one control. Beyond the fragility, **this cannot be ported** — there is no DOM in React
Native, so this logic has to be rewritten from scratch rather than translated.

**5 — BreezeGuard validates one field at a time, in the wrong order.** Instead of collecting all errors like
PostPro and TWD do, it checks sequentially and returns on the first failure — and it checks **terms and
conditions first**. So a user who has filled in nothing at all is told only "please accept the terms", and
then has to press the button once per remaining field to discover them one at a time: panel, then
stabiliser, then wind speed, then factor of safety. Five presses minimum to learn what the form wants.

**6 — BreezeGuard repeats the phantom-price bug, and makes it worse.** All four answers are sent to the
basket as priced options at **£99.00 each** flagged `oneOff` — and the wind-speed option is sent with
**`qty` set to the wind speed itself**. A site at 25 m/s produces an option with quantity 25 at £99.00.

The **amount charged is unaffected** (the line total comes from the product's own extra-price field, which
is zero). But the order-email builder routes every `oneOff` option into "Added Extras" with its price and
`price × qty` as a total — so a BreezeGuard order confirmation is likely to show four phantom extras, one of
them at 25 × £99.00 = **£2,475**, against a correctly-charged £99.00. Verify against a real order email
before acting, but the code path is unambiguous.

**7 — TWD silently drops the what3words location at checkout.** Each modal sends its answers as basket
options identified by uid, and at order placement the backend deletes any option whose uid isn't a real
variant option on the product. I checked every uid against the database:

| Service | Result |
|---|---|
| PostPro | all 6 uids valid ✅ |
| BreezeGuard | all 4 uids valid ✅ |
| TWD | 9 of 10 valid — **`57c9c33c-32c6-42e5-8186-0b3bf4859960` ("3 words") does not exist** ❌ |

So the what3words location the customer entered is stored in the basket, shown to them, and then **deleted
when the order is placed**. Engineers never see it.

### One thing TWD gets right

TWD's options are all sent with **`price: 0`** — it does not have the phantom-extras bug. Given that
PostPro's copy-pasted price is *£749.99, TWD's own price*, the sequence is clear: **TWD was written first
and correctly, then copied twice with the price constant left in.** Worth mentioning to the team, because it
means the fix is the same three-line fix in two files.

---

## 9. What this means for the mobile app

1. **Model this as one "engineering services" section**, not two blocks. Two checkboxes, mutually exclusive,
   one action — and on 60 of 70 pages, only one checkbox exists. Consider dropping the checkbox entirely and
   showing one button per available service.
2. **Don't confuse the banner with the checkbox.** The `#twd` tag banner and the `showTempWorksDesign`
   checkbox are separate features on separate data. If you port one, decide deliberately about the other.
3. **BreezeGuard is the easy one.** No location, no map, no files — four questions and a wind number. It is
   the one of the three that could plausibly work as a bottom sheet rather than a full screen.
4. **TWD is the hardest of the three.** Location + map + a five-way branching catalogue picker + files +
   notes. And its 53 SKUs are hard-coded in the web source, so **porting it means either re-hard-coding that
   list in the app or building an API for it first.** Scope this before committing to it.
5. **Rewrite the button logic**, don't port it (bug 4).
6. **Batch the validation** (bug 5) and take prices from the API (section 7).
7. **Reconsider whether these ship at all.** 70 products, one product family, third-party services the
   business explicitly disclaims. "Call us for a Temporary Works Design" is a defensible v1 — but a
   half-completed TWD submission is useless to the engineers, so it is all or nothing.

### A note on `tags`

Both tag-driven banners (`TwdBanner` and `TradeInstallerBanner`) call `.some()` on `tags` with **no null
guard**. 1,673 products in Mongo have no `tags` key at all. This is safe on the web only because the
Mongoose schema declares `tags` as an array, so the API returns `[]` rather than null. The GraphQL type is
nullable, so **an app reading the CDN directly must not rely on that** — guard it.

---

## 10. Source references

**Frontend** (`gatsby-website`):
- `src/features/breezeguard/breeze-guard.js` — the shared card, both checkboxes, the DOM-reading button
- `src/features/breezeguard/breezeguard-modal/breezeguard-modal.js` — the wind-load form; hidden product id
  at line 30, the sequential validation at ~line 130, the four £99.00 options at ~line 167
- `src/features/breezeguard/wind-table.js` — the 21 UK wind regions
- `src/features/breezeguard/temporary-works-design-modal/temporary-works-design-modal.js` — the TWD form;
  product id at line 46, cart-then-upload at lines 164 and 213
- `src/features/breezeguard/temporary-works-design-modal/select-product.js` — 1,089 lines, 53 hard-coded SKUs
- `src/features/breezeguard/temporary-works-design-modal/processOptions.js` — correctly priced at 0; the
  invalid "3 words" uid at line 37
- `src/features/breezeguard/temporary-works-design-modal/upload-files.js` — PNG/JPEG/PDF, 10 MB
- `src/features/product/twd-banner/twd-banner.jsx` — the `#twd` tag banner
- `src/templates/product/product.js:429` (banner) and `:511` (the card);
  `src/templates/group-product/group-product.js:341`

**Backend** (`website-api`):
- `start/routes/public.js:104` — `POST /upload/twd`
- `app/Utils/product-helpers.js:16` — `removeInvalidProductOptions`, which deletes the "3 words" option
- `app/Utils/order-helpers.js:26`, `app/Utils/price-calculations.js:34`,
  `resources/views/emails/order/products.edge:98` — the "Added Extras" pricing path

**Data** (Mongo `cdn`):
- `products.showBreezeGuard` — 10 live · `products.showTempWorksDesign` — 69 live · `tags.name = "#twd"` —
  55 live
- `65cde569f86b79d2ba8a6501` BreezeGuard £99.00 hidden · `65eb1595f86b79d2ba716ab9` TWD £749.99 hidden
