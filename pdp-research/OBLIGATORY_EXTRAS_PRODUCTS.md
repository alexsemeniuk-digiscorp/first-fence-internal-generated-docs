# Products with obligatory Extras

Which products force you to buy an Extra before you can add them to the basket, why that happens,
and a list of working dev-site examples for QA.

Written for the question *"find me a product where the Extras are not optional — you cannot proceed
to purchase without them."*

Everything here is measured against the **dev content database** (`cdn-test` on
`ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`) as of **26 August 2026**, and checked against the
code that renders the product page (`gatsby-website`) and the code that stores the basket
(`website-api`). No existing file has been changed.

---

## The short answer

**There is no "mandatory" flag anywhere in the system.** Nobody ever ticked a box to say "this Extra is
required". The behaviour is an *accident of configuration*: an Extra becomes unavoidable when it is set
up as a single-choice list in which **every option costs money** and **there is no "None Required"
option**. The page auto-selects the first option, and there is no way to un-select it.

**21 published, visible, in-stock products** on dev behave this way today. The clearest example:

> **[1.6m Armco Kit Calculator](https://dev.firstfence.co.uk/1600-armco-kit/)** — the product itself is
> £0.00. The Extra *"Choose your Rail Type"* has four options, all priced, none free. Whatever you do,
> you leave the page having bought at least £35.36 of rail. The same page also has *"Add Post Bolts?"*,
> *"Add Corners?"* and *"Add Ends?"*, which each start with a **"None Required"** option — so those three
> are genuinely optional. It is a good side-by-side of both behaviours on one screen.

Use trailing slashes on dev links. `https://dev.firstfence.co.uk/1600-armco-kit` (no slash) answers
**301 to `http://`** — plain HTTP, not HTTPS. `…/1600-armco-kit/` answers 200 directly.

---

## 1. What an "Extra" actually is

"Extras" is not one thing in the data. It is four overlapping structures in the `cdn-test` Mongo
database (see `cdn-graphql-v2/src/models/`):

| Structure | Where it lives | What it is |
|---|---|---|
| **`oneOff` variant** | `productvariants` collection, `oneOff: true` | The real "Extra". A block of options presented as an add-on rather than a core configuration choice. Model comment literally says `// true -> extra option`. |
| **`upsellProducts`** | on the product record | Product-level "you might also want" items. Always optional. |
| **`globalUpsellProducts`** | on an individual option | Option-scoped "you might also want" items. Always optional. |
| **`purchasableExtras`** | on the product record | A flat list of other product IDs. No quantity rule, no requirement. |

Only the first one — the **`oneOff` variant** — can ever become obligatory. There are **728** of them in
the dev database.

---

## 2. The two rules that make an Extra unavoidable

Both rules come from the front end (`gatsby-website/src/models/ProductVariant.js`). Neither is enforced
by the API.

### Rule 1 — one option is always selected

`ProductVariant.hideOptions()` ends with:

> if nothing in this variant is selected, and the variant is **not** a `CHECKBOX`, select the first
> visible option.

And `selectOption()` only ever *sets* `isSelected = true` for a non-multiselect variant — clicking the
already-selected option does not clear it. So for every variant type except `CHECKBOX` (and except a
`PRODUCT` variant with `multiselect: true`), **exactly one option is selected at all times, and the
customer cannot get back to "none".**

The usual escape hatch that content authors add is a **free option** — "None Required", "Not Required",
"No Posts Required". It is not a feature; it is a convention. Where somebody forgot to add it, the Extra
is compulsory.

### Rule 2 — the quantity box will not go below 1 on the first option

If the variant has `qtyInput: true`, each selected priced option gets a quantity box. Its minimum is set
in `gatsby-website/src/features/product/variants/product/option.js`:

```js
min={isFirstOption ? 1 : 0}
```

So the **first** option in the list — which is also the one auto-selected by Rule 1 — cannot be typed
down to zero. Options further down the list can.

### The resulting two tiers

| Tier | Configuration | Can the customer avoid it? |
|---|---|---|
| **Hard-forced** (21 products) | `oneOff: true`, not a checkbox, every option priced, **`qtyInput: false`** | **No.** There is no quantity box at all. One option is always selected, always at quantity ≥ 1. |
| **Near-forced** (32 products) | Same, but **`qtyInput: true`** | Only by a non-obvious workaround: select an option that is *not* first in the list, then type its quantity down to 0. The default selection cannot be zeroed. |

For QA purposes the **hard-forced** list below is the airtight one.

---

## 3. The QA list — 21 hard-forced products (dev)

All are `PUBLISHED`, `visible: true`, `inStock: true`, and all 21 URLs return HTTP 200 on dev.

| Product (dev link) | SKU | Base price | Obligatory Extra | Options | Cheapest you can escape with |
|---|---|---|---|---|---|
| [1.6m Armco Kit Calculator](https://dev.firstfence.co.uk/1600-armco-kit/) | `1600-ARMCO-KIT` | £0.00 | **Choose your Rail Type** | 4 | £35.36 |
| [3.2m Armco Kit Calculator](https://dev.firstfence.co.uk/3200-armco-kit/) | `3200-ARMCO-KIT` | £0.00 | **Choose your Rail Type** | 4 | £65.84 |
| [Temporary Fencing Calculator](https://dev.firstfence.co.uk/temporary-fencing-calculator/) | `WEB-08357` | £0.00 | **Pick your Temporary Fencing Panels** | 3 | £27.55 |
| [FirstShield® Tree Protection Temporary Fencing Panel Frame](https://dev.firstfence.co.uk/firstshield-tree-protection-temporary-fencing-panel-frame/) | `WEB-08494` | £0.00 | **Select Your Kit Style** | 2 | £185.17 |
| [FirstShield® Tree Protection Temporary Panel Frame Kit - Timber](https://dev.firstfence.co.uk/firstshield-tree-protection-temporary-panel-frame-kit-timber/) | `WEB-08495` | £35.90 | **Pick your Temporary Fencing Panels** | 2 | £27.55 |
| [Agricultural Fencing Staples - 1kg Box](https://dev.firstfence.co.uk/agricultural-fencing-staples/) | `FF-WEB-0008` | £4.64 | **Staple Size** | 5 | £5.57 |
| [Knee Rail Fencing with 1.8m Top Rail - Calculator](https://dev.firstfence.co.uk/knee-rail-calculator/) | `WEB-03895` | £0.00 | **Choose your top rail size** | 2 | £9.09 |
| [Knee Rail Fencing with 3.6m Top Rail - Calculator](https://dev.firstfence.co.uk/knee-rail-fencing-with-3-6m-top-rail-calculator/) | `WEB-04895` | £0.00 | **Choose Your Top Rail Size** | 2 | £15.71 |
| [Recycled Plastic Knee Rail Fencing Calculator - 0.9m and 1.2m System](https://dev.firstfence.co.uk/recycled-plastic-knee-rail-fencing-calculator/) | `WEB-08308` | £0.00 | **Choose Your Top Rail** | 2 | £50.11 |
| [StrainSure® Stock Fence \| High Tensile](https://dev.firstfence.co.uk/stock-fence-high-tensile/) | `WEB-TENS-0000` | £0.00 | **Choose your StrainSure specification** | 6 | £62.84 |
| [AntlerGuard® \| Deer Stock Fence](https://dev.firstfence.co.uk/deer-stock-fence/) | `WEB-ANTLER-CALC` | £0.00 | **Choose your AntlerGuard specification** | 2 | £197.23 |
| [EquiMesh® \| Stock Fence](https://dev.firstfence.co.uk/horse-stock-fence/) | `WEB-EQUI-0000` | £0.00 | **Choose your EquiMesh Specification** | 3 | £127.92 |
| [PastureGuard® Stock Fence](https://dev.firstfence.co.uk/pastureguard-stock-fence-calc/) | `WEB-PASTURE-CALC` | £0.00 | **Choose Wire Specification** | 11 | £27.01 |
| [AgriHex® Fencing Calculator](https://dev.firstfence.co.uk/agrihex-fencing-calculator/) | `WEB-HEX-0001` | £0.00 | **Choose your specification** | 14 | £43.58 |
| [Plain Round Black & Yellow Bollard](https://dev.firstfence.co.uk/plain-round-black-yellow-bollard/) | `WEB-BOLLHIVIS-0001` | £0.00 | **Choose your Diameter** | 48 | £106.47 |
| [Concrete Posts to Suit 0.9m Chain Link Fencing System](https://dev.firstfence.co.uk/concrete-posts-to-suit-0-9m-chain-link-fencing-system/) | `WEB-04198` | £16.80 | **Choose Your Post Style** | 4 | £16.8 |
| [Concrete Posts to Suit 1.2m Chain Link Fencing System](https://dev.firstfence.co.uk/concrete-post-to-suit-1-2m-chain-link-fencing-system/) | `WEB-04194` | £21.60 | **Choose Your Post Style** | 4 | £21.6 |
| [Concrete Posts to Suit 1.5m Chain Link Fencing System](https://dev.firstfence.co.uk/concrete-posts-to-suit-1-5m-chain-link-fencing-system/) | `WEB-04195` | £32.40 | **Choose Your Post Style** | 4 | £32.4 |
| [Concrete Post - to Suit 1.8m Chain Link Fencing System](https://dev.firstfence.co.uk/concrete-post-to-suit-1-8m-chain-link-fencing-system/) | `WEB-04196` | £40.80 | **Choose Your Post Style** | 4 | £40.8 |
| [Cranked Concrete Post - to Suit 1.8m Chain Link Fencing System](https://dev.firstfence.co.uk/cranked-concrete-post-to-suit-1-8m-chain-link-fencing-system/) | `FF-WEB-0075` | £50.98 | **Choose Your Post Style** | 5 | £77.12 |
| [Cranked Concrete Post - to Suit 2.4m Chain Link Fencing System](https://dev.firstfence.co.uk/cranked-concrete-post-to-suit-2-4m-chain-link-fencing-system/) | `FF-WEB-0081` | £70.86 | **Choose Your Post Style** | 5 | £97.49 |

### Which three to hand to QA first

1. **[1.6m Armco Kit Calculator](https://dev.firstfence.co.uk/1600-armco-kit/)** — £0 base product, one
   obligatory Extra (*Choose your Rail Type*) sitting next to three genuinely optional ones. Best single
   page for demonstrating the difference.
2. **[FirstShield® Tree Protection Temporary Panel Frame Kit - Timber](https://dev.firstfence.co.uk/firstshield-tree-protection-temporary-panel-frame-kit-timber/)**
   — one hard-forced Extra (*Pick your Temporary Fencing Panels*) **plus** two near-forced ones
   (*Add Post Kit*, *Pick your End Post Kit*). Good for testing the tier-2 behaviour in the same session.
3. **[Agricultural Fencing Staples - 1kg Box](https://dev.firstfence.co.uk/agricultural-fencing-staples/)**
   — the cheapest and simplest case: a £4.64 product where *Staple Size* forces a minimum £5.57 spend.
   Useful when you want a small basket total.

### A quirk worth noticing on the Concrete Post rows

On the five concrete-post products the obligatory Extra (*Choose Your Post Style*) has a first option
priced **identically to the product's own base price** (e.g. `WEB-04198` is £16.80 and *Intermediate
Post* is £16.80). These read less like an "Extra" and more like a product picker that was modelled as an
Extra. Worth a separate look — a customer choosing "Intermediate Post" may be paying twice, once for the
base product and once for the Extra. **Not verified end-to-end in the basket; flagging only.**

---

## 4. The 32 near-forced products (tier 2)

Same shape, but `qtyInput: true`, so a determined customer can escape by picking a non-first option and
zeroing it. Grouped by family:

- **Twin Mesh Security Fencing Kits** (9 products, `/1-2m-high-868-twin-mesh-security-fencing/` and
  siblings) — Extra *"Privacy Infill Colour"*, six options, all £40.59, no free option.
- **Timber Strainer Post Kits** (22 products, `/tanalised-timber-*-strainer-post-kit-*/` and
  `/creosoted-timber-*-strainer-post-kit-*/`) — Extra *"Choose your fencing"*, ~40 options, all priced.
- **[Timber Construction Site Hoarding Calculator](https://dev.firstfence.co.uk/timber-construction-site-hoarding-calculator/)**
  (`FF-WEB-0130`).

---

## 5. A different mechanism that also blocks checkout

Worth knowing about because QA may hit it and mistake it for the same thing.

An **option** (not necessarily an Extra) can carry a `minQuantity`. If the selected option's
`minQuantity` is higher than the quantity in the box, `getErrors()` in
`gatsby-website/src/utils/product/validation.js` blocks Add to Basket with:

> *Minimum order quantity is 50 for the selected option.*

Exactly **one** product in the dev database uses this:
**[Galfan 358 Mesh Sheets - Panels Only](https://dev.firstfence.co.uk/358-galfan-mesh-sheets/)**
(`WEB-01393`, £108.28). Its *Height* switch carries per-option minimums from 10 up to 50 sheets. This is
a **minimum order quantity**, not a forced Extra — the product is `oneOff: false`.

This matters beyond QA: **`minQuantity` is currently the only signal the business has for "this is
required"**, and it is used for two completely different things. See section 6.

---

## 6. Things worth checking / raising

### 6.1 The known gap — there is no mandatory flag

This is already logged as backend work but has not landed:

- `website-api/.generated_docs/epic-3-checkout/FIR-16-Cart-Session-Refactor-Strategy.md` §6 — a code
  search across `cdn-graphql-v2/src` for `mandatory`, `required`, `autoAdd`, `nonRemovable`,
  `removable`, `preselect`, `defaultSelected` returns **zero hits**.
- `website-api/.generated_docs/epic-2-products/FIR-58-PDP-API-Backend-Assessment.md` gaps #10 and #20 —
  *"Mandatory vs recommended extras flag — not modelled"*, sized **S**.
- `ff-uk-mobile/docs/ARCHITECTURE.md:235` lists the same thing as an open item for the app.
- `FirstFence-Meetings.md` (lines 422-436) — design wants a disabled toggle reading *"Required for this
  product"*, and First Fence still owe the list of which Extras are truly mandatory.

The findings in this document are the practical answer to *"which Extras are mandatory today?"* — the
21 products in section 3, all of them mandatory **by accident of configuration**, not by intent.
Somebody at First Fence should confirm whether each of the 21 is *meant* to be compulsory. If any is
not, the fix is a content fix (add a free "None Required" option), not a code fix.

### 6.2 Nothing on the server enforces any of this

`CartController` in `website-api` does not validate Extras at all. The add-to-basket validator
(`app/Validators/StoreProduct.js`) treats `product.options` as an optional array:

```js
"product.options": "isArray",
```

There is no check that a required Extra is present, and no check of `minQuantity`. Both gates live
entirely in the Gatsby front end. Two consequences:

- **A basket built by anything other than the website can skip an obligatory Extra.** Relevant to the
  mobile app (FIR-16 / FIR-18 work) and to any direct API testing — QA can create a basket state the
  website cannot produce.
- The same validator takes `product.priceEa` and `product.extraPrice` **from the client payload**, so
  prices are client-asserted at the point the basket line is written. Worth a separate look by whoever
  owns checkout hardening; out of scope for this note.

### 6.3 Hidden linked products inside Extras

On the Armco kits, most options under *"Post Height"* link to products with `visible: false` (e.g.
*Armco 560mm Galvanised RSJ Post*). They still render and are still purchasable through the Extra. Not a
bug as such, but it means a hidden product can reach the basket, which may matter for stock, pricing-tier
and SAP-SKU testing.

---

## 7. How this was measured (repro)

Connect to the dev content database:

```bash
docker exec -i firstfence-mongo mongosh --quiet \
  --host ec2-34-252-129-247.eu-west-1.compute.amazonaws.com --port 27017 \
  -u firstfence -p '<MONGO_DB_PASSWORD from website-api/.env>' \
  --authenticationDatabase cdn-test cdn-test
```

Note: `--authenticationDatabase cdn-test`, **not** `admin` — `authSource=admin` fails. The credentials
are in `website-api/.env`, in the commented-out "external test Mongo" block.

The classification query, in words:

1. Take every `productvariants` document with `oneOff: true`.
2. Drop `variantType: "CHECKBOX"` and `variantType: "PRODUCT"` with `multiselect: true` — those can be
   un-ticked.
3. For each remaining variant, resolve every option's effective price: `priceModifier` if set, otherwise
   the `price` of the product named by `productLink`.
4. Keep the variant only if **every** option resolves to a price greater than zero — i.e. there is no
   free "None Required" escape.
5. Split on `qtyInput`: `false` → hard-forced, `true` → near-forced.
6. Join back to `products` on `productVariants` and filter to `status: "PUBLISHED"`, `visible: true`.

Result: **63 forced variants**, used by **73 products**, of which **21** are published + visible +
hard-forced, and **32** are published + visible + near-forced. The rest are `TEST`, `DRAFT`, `ARCHIVED`
or hidden.

Numbers in this document describe the dev database. Production may differ.
