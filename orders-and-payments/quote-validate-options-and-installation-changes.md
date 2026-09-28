# Quote conversion: how to get `optionsChanged.removed`, and why "Installation Not Available" sends no change

QA is testing quote-to-order conversion in the mobile app. They asked two questions (translated
from Ukrainian, sent as is):

> 1. How do I make the backend return `optionsChanged.removed`? When I archive, it responds with
>    `notAdded`.
> 2. Can I remove an installation option in the admin? I want to test quote conversion where the
>    quote has installation, but then the product becomes unavailable for installation. I think I
>    found it in the admin — is this the right thing (screenshot: product edit page, field
>    **Installation Type** = **Installation Not Available**)? After converting the quote to an
>    order, the backend sends nothing in the changes. The quote has installation; the PDP shows no
>    installation.

**Data provenance:** everything was measured on **2026-09-28**, read-only. Content data from the
**dev** content database (`cdn-test` @ `ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`).
Quote, cart and order data from the **dev** MySQL database (`website_test` @ `54.171.181.199`).
Code read at `website-api` **`9e3da5d`** (branch `mobile-app`; the files that matter here are
identical on `origin/dev-docker` **`527fba9`**, the branch that deploys to dev), `cdn-graphql-v2`
**`0cf6608`** (`mobile-app`), `admin-website-v2` **`c4287e6`** (`main`), `ff-uk-mobile`
**`15aefd9`** (`main`). No writes of any kind were made. Production was not touched. The dev API
was not called (the `validate` endpoint needs QA's customer login); instead, the backend's checks
were replayed against the live dev data (§4.3, Appendix A).

Companion docs: [`installation-price-in-basket.md`](./installation-price-in-basket.md) (the
`surcharges.length > 1` bug, §6.1), [`order-and-payment-status-flow-trade-credit.md`](./order-and-payment-status-flow-trade-credit.md)
(what the quote-conversion order looks like after it is created).

---

## TL;DR

**Question 1.** Archiving the **product** gives `notAdded` on purpose. When a product is not
available, the backend stops early and checks nothing else on that line. To get
`optionsChanged.removed`, keep the product published and remove an **option** the quote used (or
unlink its variant from the product). Section 3 has the exact steps for QA's own quote 3909.

**Question 2.** Yes, the screenshot field is the right real-world setting. It is the **only** place
in the admin that says whether a product offers installation, and the PDP obeys it. **But the
backend does not read it.** The quote check (`validate`) only asks "does the installation type
document still exist, and does its surcharge count still match?" Switching a product to
"Installation Not Available" changes neither, so `changes` stays empty. QA did nothing wrong. This
is a **gap in `website-api`** (§4).

Three more facts QA and the team need:

| # | Fact | Whose fix |
|---|---|---|
| 1 | Only `POST /customers/quotes/:id/validate` returns `changes`. `convert-to-order` never returns them. | — (by design) |
| 2 | `convert-to-order` **copies the quote's installation onto the order and charges the quote's installation price anyway**. If QA converts quote 3909 now, the order will charge **£4,920** installation for a product that no longer offers it. | `website-api` (backend) |
| 3 | The mobile app **ignores `changes` for quotes**. It reads only `valid`. Even when the backend does report a change, the customer sees nothing. QA can only see changes in the network log. | `ff-uk-mobile` (FIR-131) |

---

## 1. The short answer

| Question | Answer |
|---|---|
| Why does archiving give `notAdded`? | An archived product is "not available". The backend returns `notAdded: "unavailable"` and skips every other check on that line (`reorder-changes.js:198-200`). |
| How do I get `optionsChanged.removed`? | Product stays PUBLISHED and visible. Remove one of the quoted options from its variant, or unlink the variant from the product. Then call `validate` again (§3). |
| Is the screenshot field the right one for "no installation"? | **Yes**, for the real business case. It writes `products.installationType = null`. |
| Why did the backend send nothing? | The backend never reads `products.installationType` when it checks a quote (§4.2). Replaying the check on quote 3909 gives `changes: []` (§4.3). |
| Can I make the backend report `installationUnavailable` today, from the admin? | Only in an indirect way: change the **surcharge count** of the installation type, or delete the type. Both are risky on the shared type (§4.5). A safer way with a test type is in §4.5. The better answer is the backend fix (§7). |
| Does `convert-to-order` return changes? | **No.** Only `validate` does (`CustomerQuoteController.js:325-384`). |
| Will the app show a change to the customer? | **No, not for quotes.** `PayForQuoteScreen.tsx:63-67` reads only `valid` (§5). |

---

## 2. How the backend builds `changes`

`POST /customers/quotes/:id/validate` (`start/routes/customer-portal.js:74`,
`CustomerQuoteController.validateForConversion`, `:325`) loads the quote lines and calls
`ReorderValidationService.revalidateLines` (`:367`). This is the **same code as mobile reorder**
(FIR-157). For each quote line, `evaluateLine` (`app/Utils/reorder-changes.js:179`) runs these
checks **in this order**:

| Step | Check | Result | Code |
|---|---|---|---|
| 1 | Line has no `mongo_id` | `notAdded: "no_catalogue_link"`, **stop** | `reorder-changes.js:194-196` |
| 2 | Product missing, `deleted`, hidden (not a kit child), or `status !== "PUBLISHED"` | `notAdded: "unavailable"`, **stop** | `:198-200`, rule at `:70-81` |
| 3 | Line is a hire product | `notAdded: "hire_requires_new_dates"`, **stop** | `:205-207` |
| 4 | `inStock` changed | `stockChange` | `:213-217` |
| 5 | Lead time changed | `leadTimeChange` | `:219-226` |
| 6 | Qty above `orderLimitQuantity` / below `minQuantity` | `qtyCapped` / `qtyRaised` | `:228-238` |
| 7 | A stored option `uid` is not on the product's variants any more | `optionsChanged: { removed: [uid…] }` | `:240-250` |
| 8 | `getInstallationType` returns `null` | `installationUnavailable: true` | `:252-258` |

A line with no facet set gives no entry. An unchanged quote returns `changes: []`.

```js
// reorder-changes.js:198-200
  if (!isAvailable(doc, { soldViaLiveKit })) {
    return notAdded(NOT_ADDED.UNAVAILABLE);
  }
```

The early `return` in steps 1-3 is why archiving never shows `optionsChanged`. A line that is not
added has no options left to compare.

---

## 3. Question 1 — how to get `optionsChanged.removed`

### 3.1 What the backend compares

`resolveVariantOptionUids` (`ReorderValidationService.js:147-153`) calls `getAllVariantOptions`
(`app/Utils/product-helpers.js:77-107`). That function:

1. reads the product's `productVariants` id list from Mongo `products`;
2. loads those documents from `productvariants`;
3. collects every `options[].uid`.

Any option `uid` stored on the quote line (`product_options.uid`) that is **not** in that set is
reported as removed. There is no status or `deleted` flag on variants or options at all
(`cdn-graphql-v2/src/models/product-variants.js:3-46`; 0 of 4,564 dev variants have `deleted` or
`status`), so "archiving" a variant is not possible. The option must really leave the list.

### 3.2 Option `uid`s are stable — measured

If the admin created new `uid`s on every save, every edit would look like "removed". It does not:

| Evidence | Result |
|---|---|
| Code | New/duplicated options get a client-side `uuidv4()` (`admin-website-v2/src/pages/content/product-variants/product-variant-options/product-variant-options.jsx:37,100`). Save keeps existing `uid`s. **Copy Variant** creates a new document with all-new `uid`s. |
| Dev audit log `productvariantlogs` (1,265 logged saves with options, 2025-11-21 → 2025-12-17) | **1,227** kept every `uid`. **31** lost a `uid` because an option was really deleted. **7** lost a `uid` but still had an option of the same name (deleted and added again → new `uid`). |

So "delete an option, then add it back with the same name" still counts as removed. The old `uid`
never comes back.

### 3.3 Recipe for QA's quote 3909

QA's open quote **3909** (created 2026-09-28 13:25:54 UTC) has two lines of
*1.8m High 'W' Section Palisade Security Fencing Kit* (`5f16f3be8b49590d02dea1f6`), each with the
same 4 options:

| Quoted option | `uid` (start) | Variant | Products using this variant | Safe to change? |
|---|---|---|---|---|
| Post Type: **Dig In** | `5a63fe44` | `698d96b72d4a743dba6aa80a` | 1 (this product only) | **Yes** |
| Pale Type: **Triple Point Pales** | `670940fc` | `698d945a87a796f378afac58` | 1 | Yes |
| End post: **Dig In End Post** | `75ff0cd2` | `5f16ad1ef0eb59044ceadba8` | 1 | Yes |
| Add Corner Fixing Kit: **Corner Fishplate Kit** | `f5f99016` | `6393233fed66044511f2b260` | **16** (15 published) | **No — shared** |

Two ways to do it, best first:

| Way | Admin steps | Result on quote 3909 | How to undo |
|---|---|---|---|
| **A. Unlink the variant (recommended)** | Products → edit the 1.8m kit → **Variants** tab → trash on variant `698d96b7…` (Post Type) → **Save** | `optionsChanged: { removed: ["5a63fe44-…"] }` on both lines | Add the same variant back → **the same `uid`s return**. Fully reversible. |
| B. Delete the option | Product Variants → edit `698d96b7…` → **Options** tab → trash "Dig In" → **Save** | Same result | **Not reversible.** Adding "Dig In" again creates a new `uid`, so every old quote and order with that option will report it as removed forever. |

Do **not** delete a whole variant (Product Variants → trash). It is a hard delete
(`cdn-graphql-v2/src/resolvers/product-variant.js:173-190`, `ProductVariant.remove()`), it leaves a
broken id in every product's `productVariants`, and it cannot be undone.

After the change, call `validate` on quote 3909 again. The product itself must stay PUBLISHED and
visible, or you get `notAdded` again.

---

## 4. Question 2 — "Installation Not Available" and why nothing comes back

### 4.1 What the screenshot field does

| Step | Evidence |
|---|---|
| The dropdown's first item is `{ id: '', value: 'Installation Not Available' }` | `admin-website-v2/src/pages/content/product/main-fields/main-fields.jsx:43-50` |
| On save, `installationType: node.installationType?.id \|\| null` → `''` becomes `null` | `admin-website-v2/src/pages/content/product/normalize.js:114` |
| Mongo field `products.installationType` (id stored as a string) | `cdn-graphql-v2/src/models/product.js:96` |
| On dev now: the 1.8m kit has `installationType: null`. Its sister kits (2.0m, 2.1m, 2.4m, 3.0m) still have `62ecec379f11e1c565a59c64` ("Palisade Fencing"). | Appendix A, Q2 |
| No other admin screen sets a product's installation. The installation type page lists no products. | admin/CDN grep |

So QA picked the right field for the real business case.

### 4.2 What the backend checks instead

The quote line stores its own copy of the installation in MySQL: one `installation_types` row
(`mongo_id` = the Mongo type id) plus one `installation_surcharges` row per surcharge. `validate`
checks that copy with `getInstallationType` (`app/Utils/installation.js:45-101`):

```js
// ReorderValidationService.js:163-169
  async resolveInstallation(line) {
    if (!line.installationType) {
      return null;
    }
    const resolved = await getInstallationType(line.installationType);
    return !!resolved;
  }
```

It never receives the product document. The only function that reads `products.installationType`
is `isInstallationAvailable` (`installation.js:195-206`), and it is used only for order emails.

What `getInstallationType` returns in each case (validate passes `ignoreSurchargeMismatch = false`):

| Situation | `getInstallationType` | `installationUnavailable` | Code |
|---|---|---|---|
| **Product set to "Installation Not Available" (QA's case)** | object — not checked | **never** | — |
| Product switched to a **different** installation type | object — not checked | **never** | — |
| Mongo type has `deleted: true` | object (the flag is only copied) | never. The admin never writes this flag anyway — all deletes are hard. | `installation.js:81` |
| Mongo type document deleted | `null` | **yes** | `:94-96` |
| Mongo type has 3 surcharges, line stored 2 | `null` | **yes** | `:117-121`, `:185` |
| Mongo type has 2 surcharges, line stored 2 | object | no | `:117-121` |
| Mongo type has **1** surcharge | `null` — see note | **yes, even with no change (false alarm)** | `:24`, `:117-129` |
| Mongo type has **0** surcharges | `null` | **yes, even with no change (false alarm)** | `:117-129` |

Note on the 1-surcharge row: `saveInstallationType` only writes surcharge rows when
`surcharges.length > 1` (`installation.js:24`). So a type with exactly one surcharge is stored with
0 rows, and `1 !== 0` fails the count check. This is the bug already described in
[`installation-price-in-basket.md` §6.1](./installation-price-in-basket.md).

### 4.3 Replay of quote 3909 — measured

I repeated each backend check on quote 3909's 4 lines against the live dev data (Appendix A, Q3):

| Line | Product | Available | Stock / lead time / qty change | Options removed | Installation resolves | Product's `installationType` now |
|---|---|---|---|---|---|---|
| 2404520 | 1.8m W Palisade Kit | yes | none | `[]` | **true** (Mongo 8 surcharges = SQL 8 rows) | **`null`** |
| 2404521 | Rapid Set Post Mix Cement | yes | none | — | — (no installation) | `null` |
| 2404522 | 1.8m W Palisade Kit | yes | none | `[]` | **true** (8 = 8) | **`null`** |
| 2404523 | Rapid Set Post Mix Cement | yes | none | — | — | `null` |

No facet is set on any line, so the backend returns `changes: []`. **This matches exactly what QA
saw.**

### 4.4 What conversion then does with the installation

`convert-to-order` (`CustomerQuoteController.js:400`) checks ownership and expiry, then calls the
public `BasketQuoteController.convertToOrder` (`:1707`):

| Step | Evidence |
|---|---|
| The request body has no lines at all — only contact, billing and notes fields | `BasketQuoteController.js:1742-1752` |
| Quote lines are copied to a new cart, **including** `installation_types` and every `installation_surcharges` row. Nothing is re-checked. | `:1506`, `:1566-1586` ("Copy installation type + surcharges") |
| The order gets the quote's installation total | `:1860` `installationPrice: quoteData.installation_price \|\| 0` |
| That price is charged at payment | `app/Utils/price-calculations.js:99-107` (`computeOrderChargeTotal`) |
| The order email shows installation "Yes" | `resources/views/emails/order/products.edge:8` (`@if(product.hasInstallation)`) |

QA's earlier orders today prove the copy happens: orders **60877** (quote 3906) and **60878**
(quote 3908) both carry the "Palisade Fencing" installation rows on their cart lines, with
`installationPrice = 2460.00`. Both were created **before** the product was changed.

If QA converts quote **3909** now, the order will have `installationPrice = 4920.00` (the quote's
`installation_price`) on a product that no longer offers installation.

### 4.5 How QA can test `installationUnavailable` today

With the current code, the admin can only trigger it by changing the **installation type**, not
the product. All these deletes are hard deletes (`cdn-graphql-v2/src/resolvers/installation-type.js:105`,
`surcharge.js:72`).

| Way | Steps | Risk on shared dev |
|---|---|---|
| Change the surcharge count of "Palisade Fencing" | Installation → Installation Types → Palisade Fencing → Surcharges tab → add or remove one → Save | **High.** 12 products use this type. **1,726** stored lines use it (59 quote lines, 1,402 lines in open carts). Every one of them changes at once. I did not check what the cart and admin screens do with those lines (§10). |
| Delete "Palisade Fencing" | Installation Types → trash | **Do not do this.** Hard delete. It breaks 12 products and 1,726 lines and cannot be undone. |
| **Use a test type (safest)** | 1. Create a new installation type with **at least 2** surcharges and at least one installation price (the PDP hides installation with no price, `ff-uk-mobile/src/features/product/blocks/InstallationBlock.tsx:19-20`). 2. Set it on the test product. 3. Create a new quote with installation. 4. Remove one surcharge from the test type and Save. 5. Call `validate`. | Low. Only lines that use the test type are affected. Afterwards, put "Palisade Fencing" back on the product and delete the test type. |

Do **not** use a test type with 0 or 1 surcharge. It reports `installationUnavailable` before you
change anything (§4.2 table).

**Recommendation:** wait for the backend fix in §7. After it, QA's original test (the screenshot
field) is the right test, and it will work as expected.

**Please also put the 1.8m kit back** to "Palisade Fencing" after testing. Right now dev offers no
installation for it.

---

## 5. The app does not show `changes` for quotes

This part is for the mobile team. It does not change the backend answer, but it explains why the
customer would see nothing even after a backend fix.

| Fact | Evidence |
|---|---|
| The quote flow calls `validate`, then `convert-to-order`, then opens payment | `ff-uk-mobile/src/features/quotes/screens/PayForQuoteScreen.tsx:60-92` |
| It reads only `valid`: `if (!validation.valid) { refuse(validation); return; }` | `PayForQuoteScreen.tsx:63-67` |
| `changes` is kept in the type but never used; `lines` is thrown away | `quotesApi.ts:259-266` |
| The "Basket updated" sheet already has the text for both facets ("One of your selected options is no longer available…", "Installation is no longer available for this item…") | `BasketUpdatedSheet.tsx:200-209` |
| But the sheet is only fed by reorder (`reorderChangesSet`, `OrderDetailScreen.tsx:231`) | mobile grep |
| Mobile reorder already detects installation loss **on its own** from the CDN `installationType` (commit `7f959fd`, 2026-09-17) | `ff-uk-mobile/src/features/orders/utils/validateReorder.ts:19-36,64-69` |

So for **reorder**, the app already shows "Installation is no longer available". For **quotes**,
the disclosure screen (FIR-131) is not built yet.

---

## 6. Timeline

| When (UTC) | What | Source |
|---|---|---|
| 2022-11-11 | `saveInstallationType` stores surcharge rows only when `> 1` (last change to `installation.js`) | `website-api` `494d0cc` |
| 2026-03-13 | Public `convertToOrder` starts copying quote installation + surcharges to the order cart | `website-api` `f1beecf` |
| 2026-08-11 | Reorder re-validation added (FIR-157): `optionsChanged`, `installationUnavailable` | `website-api` `f3cc193` |
| 2026-08-12 | App types get `optionsChanged`, `installationUnavailable`, `/validate` | `ff-uk-mobile` `1d34d27` |
| 2026-08-31 | Logged-in quote `validate` + `convert-to-order` (FIR-175, MR !283) | `website-api` `bfda7c0` |
| 2026-09-17 | Mobile reorder detects lost installation itself | `ff-uk-mobile` `7f959fd` |
| 2026-09-18 | Mobile quote flow moves to logged-in endpoints; `changes` added to the type, not used | `ff-uk-mobile` `ac438a3` |
| 2026-09-28 12:41 | `mobile-app` merged into `dev-docker` → dev deploy | `website-api` `527fba9` |
| 2026-09-28 10:52, 12:28 | QA quotes 3906, 3908 with installation → orders 60877 (10:58), 60878 (12:29), £2,460 installation each | dev MySQL |
| 2026-09-28 **13:25:54** | QA quote **3909**: 2 kit lines with installation, `installation_price = 4920.00` | dev MySQL |
| 2026-09-28 **13:44:10** | 1.8m kit saved with `installationType: null` (product `updated`) | dev Mongo |
| 2026-09-28 13:45:56 | QA's screenshot (file name 16:45:56, Ukraine summer time = UTC+3) | screenshot |
| 2026-09-28 14:46 | No order exists for quote 3909 yet. So the empty `changes` QA saw came from `validate` on 3909. | dev MySQL |

The audit logs cannot help with this product change. `productlogs` **and** `productvariantlogs`
both stop on 2025-12-17 (`d66790a` commented out product, variant, category and cms-page logging).
The last `productlogs` rows for this product (2025-11-25) show `installationType` still
`62ecec379f11e1c565a59c64`. Installation types and surcharges were never logged.

---

## 7. What needs to change, and where

### Backend — `website-api` (ours)

**Fix 1 — report the real case.** In `ReorderValidationService.revalidateLines`, the product
document `doc` is already loaded (`:38`). Pass it to `resolveInstallation` and return `false` when
the product no longer offers the stored type:

```js
async resolveInstallation(line, doc) {
  if (!line.installationType) return null;
  const offered = doc && doc.installationType;
  if (!offered || String(offered) !== String(line.installationType.mongo_id)) return false;
  return !!(await getInstallationType(line.installationType));
}
```

| Check | Result |
|---|---|
| Extra DB cost | None. `doc` is already fetched in one query. |
| Affects reorder too? | Yes — same service. Reorder is fine: the backend then sends `installation: null` plus the facet, and the app keeps a null-installation line as it is and adds no second message (`validateReorder.ts:48`). |
| "Different type" case | Treat as unavailable. The stored ground-type and surcharge option ids belong to the old type. Dev lines in this state today: **0**. |
| Tests | Add cases next to `test/reorder-validation.spec.js:304` (the `optionsChanged` case). `evaluateLine` does not change; only the value passed as `installationResolves` changes. |

**Fix 2 — conversion must not charge for it.** Fix 1 only **reports**. `convert-to-order` still
copies and charges the installation (§4.4). Options:

| Option | What | Trade-off |
|---|---|---|
| **a. Refuse (recommended first step)** | In `convertOwnedQuoteToOrder` only, run `revalidateLines`; if any line has `installationUnavailable` (or `notAdded`), return 409 with a new `reason`, like expiry does. | Reversible, no server-side pricing. The app needs to know the new reason (its known list is `REFUSAL_REASONS`, `ff-uk-mobile/src/features/quotes/services/quotesApi.ts:253-257`). The customer is sent to re-quote. |
| b. Drop installation and re-price | Remove the line's installation rows and recompute `installationPrice` | The backend does not price installation today; the quote stores **one total** for all lines (`basket_quotes.installation_price`), and the minimum-charge rule makes per-line maths wrong (see `installation-price-in-basket.md`). Not recommended now. |

Which one is right is a **business decision** (M7 in the epic-5 docs says "the customer pays the
new price"). **Trap:** do not put the guard inside the shared `BasketQuoteController.convertToOrder`.
That is also the website's public route, which is on prod.

**Fix 3, later and in its own MR — the 0/1-surcharge false alarm (§4.2).** It is a real backend
bug, but nothing is hit by it today:

| Measure | Result |
|---|---|
| Installation types by surcharge count, dev | 32 have 8, 7 have 7, 2 have 6, 1 ("Unknown") has 0. **None has exactly 1.** |
| Same, local production dump (newest product edit 2026-06-17) | Identical. "Unknown" is used by 0 published products in both. |
| All **7,873** stored `installation_types` rows on dev (carts, quotes, orders) | **7,847** resolve. **26** fail, and none because of this bug: 25 are 2022 rows saved before their types got an 8th surcharge (a real config change), 1 has `mongo_id = '1'` (2026-08-25). |
| Quote lines with installation on dev | 285; **0** false alarms. |

So the bug only starts to matter the day someone creates a type with exactly one surcharge.

| Part | Change | Trap |
|---|---|---|
| Write path | `installation.js:24`: `surcharges.length > 1` → `> 0` | Safe now. No existing row was saved wrong (no 1-surcharge type exists), so there is nothing to backfill. |
| Read path | `getInstallationSurcharges` (`:117-129`) needs `surchargeIds.length >= 1`, so a 0-surcharge type never resolves | Also used by carts (`CartController.js:80-85` swaps in the product's default type from Mongo when it fails), order emails and admin. Test those paths too. Low value today: the only 0-surcharge type is unused. |

`installation.js` has not changed since 2022 (`494d0cc`). Keep this out of the Fix 1 / Fix 2 MR.

### Mobile — `ff-uk-mobile` (not ours)

- Show `changes` from `validate` before converting (FIR-131). The sheet text already exists (§5).
- Or detect lost installation from the CDN, as reorder already does (`7f959fd`).
- `docs/ARCHITECTURE.md:200` says Quotes are **mock** (`quotes.mock`). That file was deleted on
  2026-08-07 (`aaeede8`). `:198` says reorder has "no server-side re-validation yet". Both are out of
  date.

---

## 8. Is this just dev?

| Question | Answer |
|---|---|
| **Can a deploy or pipeline cause it on prod?** | It is not caused by a deploy or by data. It is how the code works. CI: `master` → production (`.gitlab-ci.yml:4-9`), `dev-docker` → dev (`:108-118`). The mobile `validate` / `convert-to-order` code (`reorder-changes.js`, `CustomerQuoteController.js`) is **not on `master`** (checked on local refs fetched 2026-09-28 14:34 UTC). When `mobile-app` reaches `master`, prod gets the same gap. |
| **Can it happen again after the fixes already in the repo?** | **Yes.** No branch has a fix. Any admin who sets "Installation Not Available" on a product that is in an open quote causes it. On dev, **658** published products offer installation. |
| **Is prod already in that state?** | Partly, and I cannot measure it. The mobile `validate` endpoint does not exist on prod. But the website's public `convert-to-order` on `master` has copied quote installation the same way since `f1beecf` (2026-03-13). Whether prod has open quotes on products that no longer offer installation needs the prod query below. |

**Command for you to run on prod (read-only).** Step 1, prod MySQL — open quotes with installation:

```sql
SELECT p.basket_quote_id, bq.created_at, p.id AS line_id, p.mongo_id AS product,
       it.mongo_id AS installation_type, bq.installation_price
FROM products p
JOIN installation_types it ON it.product_ID = p.id
JOIN basket_quotes bq      ON bq.id = p.basket_quote_id
WHERE bq.purchase_completed = 0
  AND bq.created_at >= NOW() - INTERVAL 5 DAY   -- basket quotes live 5 days (quote-validity.js:41)
ORDER BY bq.created_at DESC;
```

Step 2, prod Mongo `cdn` — paste the `product` / `installation_type` pairs from step 1:

```js
const pairs = [ /* ['<product mongo_id>', '<installation_type mongo_id>'], ... */ ];
pairs.forEach(([pid, tid]) => {
  const p = db.products.findOne({ _id: ObjectId(pid) }, { title: 1, installationType: 1, status: 1 });
  const offered = p && p.installationType ? String(p.installationType) : null;
  if (offered !== tid) print('AFFECTED', pid, p && p.title, 'quote type', tid, 'product now', offered);
});
```

Any `AFFECTED` line is a quote that would convert with an installation the product no longer
offers. Salesforce quotes use a different expiry rule (`quote-validity.js` `resolveExpiry`); widen
the date filter if you want them too.

---

## 9. How this fits with existing docs

| Doc | Status |
|---|---|
| `website-api/.generated_docs/epic-3-checkout/FIR-21-FIR-157-Reorder-Backend-Strategy.md:531-533` | **Correct for what it covers** ("returns `null` when the Mongo type or its surcharges no longer match … emit `installationUnavailable`"). It never considered "product no longer offers installation", and it misses the 0/1-surcharge false alarm. No doc records a decision on this case. |
| `website-api/.generated_docs/epic-5-orders/linear-guides-snapshot-2026-09-16/after/05-quote-conversion.md:93` | **Wrong.** "You submit the conversion; the backend totals what you send." `convert-to-order` accepts no lines (`BasketQuoteController.js:1742-1752`) and totals the stored quote rows. The same claim is in the code comment at `CustomerQuoteController.js:321-323`. |
| same doc `:101` | **Wrong key names.** It lists `item_removed`, `stock_changed`, `lead_time_changed`, `qty_capped`. The real keys are `notAdded`, `stockChange`, `leadTimeChange`, `qtyCapped`, `qtyRaised`, `optionsChanged`, `installationUnavailable` (`reorder-changes.js:100-109`). |
| same doc `:113` | **Misleading.** Marks re-validation ✅ for the conversion endpoints. Only `validate` re-validates; `convert-to-order` does not. |
| `ff-uk-mobile/docs/ARCHITECTURE.md:198,200` | **Out of date** (§7). |
| [`installation-price-in-basket.md`](./installation-price-in-basket.md) §6.1 | **Still correct.** `installation.js:24` still has `> 1`. This doc adds one more effect: the quote check reports a false `installationUnavailable` for 1-surcharge types. |
| [`../pdp-research/MIN_MAX_QUANTITY_EXPLAINED.md`](../pdp-research/MIN_MAX_QUANTITY_EXPLAINED.md) `:313-316` | **Substance still correct; line numbers moved.** It cites `reorder-changes.js:204-213` and `:210`. After `d200f4f` (2026-09-23, FIR-231) these are now `:228-238` and `:235`. |
| [`order-and-payment-status-flow-trade-credit.md`](./order-and-payment-status-flow-trade-credit.md) `:137` | **Line numbers moved.** It cites `BasketQuoteController.js:1795-1814`; the order create is now at `:1841-1866`. The substance (`status: STATUS_CREATED`, `basket_quote_id` set) is unchanged. |

---

## 10. What I did not verify

| Item | Why |
|---|---|
| The actual `validate` response QA received | I did not call the dev API; it needs QA's login. §4.3 is my replay of the code against live data. It matches QA's report, but it is a replay, not the response itself. |
| That QA looked at `validate` and not `convert-to-order` | Inferred: no order exists for quote 3909, and only `validate` returns `changes`. |
| What carts, admin quote screens and emails do if the surcharge count of "Palisade Fencing" changes | Not traced. This is why §4.5 advises against changing the shared type. |
| The admin form for creating an installation type (fields, whether "copy" exists) | Not opened. §4.5 steps 1-2 are from the data model, not from the form. |
| That the dev box really runs `origin/dev-docker` HEAD | `redeploy.sh` is outside the repo. Supported by QA getting `notAdded`, which exists only in `reorder-changes.js` (not on `master`). |
| Prod | Not touched. See §8 for the command. |
| Who switched the product's installation off | Inferred to be QA from the screenshot time. Product logging has been off since 2025-12-17, so there is no user id. |

---

## Appendix A — queries used

Helpers: dev MySQL through `docker exec -e MYSQL_PWD=… firstfence-mysql mysql -h 54.171.181.199 -u mobile_dev website_test -e "…"`.
Dev Mongo through `docker exec -i firstfence-mongo mongosh --quiet --host ec2-34-252-129-247.eu-west-1.compute.amazonaws.com --port 27017 -u firstfence -p … --authenticationDatabase cdn-test cdn-test --eval "$(cat file.js)"`.

**Q1 — find the screenshot product (Mongo)**

```js
db.products.find({$or:[{title:/Palisade Security Fencing Kit/i},{historicMerchantCentreCode:'0340-001-040'}]},
  {title:1,status:1,productType:1,visible:1,installationType:1,productVariants:1})
```

**Q2 — product edit time and installation type (Mongo)**

```js
db.products.findOne({_id:ObjectId('5f16f3be8b49590d02dea1f6')},{updated:1,installationType:1,status:1})
db.installationtypes.findOne({_id:ObjectId('62ecec379f11e1c565a59c64')})   // 8 surcharges, deleted:false
db.products.countDocuments({installationType:'62ecec379f11e1c565a59c64'})  // 12
```

**QA's quotes and orders today (MySQL)**

```sql
SELECT id, purchase_completed, installation_price, total_price, created_at
FROM basket_quotes WHERE user_id=2392 AND created_at >= '2026-09-28';
SELECT id, basket_quote_id, cart_ID, installationPrice, created_at
FROM orders WHERE user_id=2392 AND created_at >= '2026-09-28';
SELECT o.id, p.id, p.name, it.mongo_id,
       (SELECT COUNT(*) FROM installation_surcharges s WHERE s.installation_type_ID=it.id) n_sur
FROM orders o JOIN products p ON p.cart_ID=o.cart_ID
LEFT JOIN installation_types it ON it.product_ID=p.id WHERE o.id IN (60877,60878);
```

**Quote 3909 lines, options, surcharges (MySQL)**

```sql
SELECT p.id, p.mongo_id, p.qty, p.inStock, p.leadTime, p.leadTimeDuration, o.uid, o.variantName, o.name
FROM products p LEFT JOIN product_options o ON o.product_ID=p.id
WHERE p.basket_quote_id=3909 ORDER BY p.id, o.id;
SELECT it.id, it.product_ID, CAST(s.mongo_id AS BINARY)
FROM installation_types it JOIN installation_surcharges s ON s.installation_type_ID=it.id
WHERE it.product_ID IN (SELECT id FROM products WHERE basket_quote_id=3909);
```

**Q3 — replay of `revalidateLines` for quote 3909 (Mongo)**

```js
// lines[] filled from the MySQL query above
for (const l of lines) {
  const d = db.products.findOne({_id: ObjectId(l.mongo)});
  const avail = !!d && d.deleted !== true && d.visible !== false && d.status === 'PUBLISHED';
  const curLt = d.leadTime ? (d.leadTimeDuration||0) : 0;
  let removed = null;
  if (l.uids.length) {
    const valid = new Set();
    db.productvariants.find({_id:{$in:(d.productVariants||[]).map(x=>ObjectId(x))}})
      .forEach(v => (v.options||[]).forEach(o => valid.add(o.uid)));
    removed = l.uids.filter(u => !valid.has(u));
  }
  let instResolves = null;
  if (l.inst) {
    const it = db.installationtypes.findOne({_id: ObjectId(l.inst)});
    instResolves = !!it && (it.surcharges||[]).length >= 1 && (it.surcharges||[]).length === l.sqlSur;
  }
  printjson({line:l.id, available:avail, stockChange:l.inStock !== (d.inStock !== false),
    leadTimeChange:l.lt !== curLt, optionsRemoved:removed, installationResolves:instResolves,
    productInstallationTypeNow:d.installationType});
}
```

**Q4 — option `uid` stability in `productvariantlogs` (Mongo)**

```js
let n=0, same=0, sameName=0, real=0;
db.productvariantlogs.find({'previous.options.0':{$exists:true},'current.options.0':{$exists:true}}).forEach(l => {
  n++;
  const cu = new Set(l.current.options.map(o=>o.uid)), cn = new Set(l.current.options.map(o=>o.name));
  const lost = l.previous.options.filter(o=>!cu.has(o.uid));
  if (!lost.length) same++; else if (lost.every(o=>cn.has(o.name))) sameName++; else real++;
});
// n=1265 same=1227 sameName=7 real=31; log range 2025-11-21 → 2025-12-17
```

**Q5 — which variant holds each quoted option, and how many products share it (Mongo)**

```js
for (const vid of p.productVariants) {
  const v = db.productvariants.findOne({_id:ObjectId(vid)});
  print(vid, db.products.countDocuments({productVariants: vid}), v.options.map(o=>o.name+' '+o.uid));
}
```

**Q6 — every quote line with installation on dev, replayed (MySQL → Mongo)**

```sql
SELECT p.id, p.basket_quote_id, bq.created_at, bq.purchase_completed, p.mongo_id, it.mongo_id,
       (SELECT COUNT(*) FROM installation_surcharges s WHERE s.installation_type_ID=it.id)
FROM products p JOIN installation_types it ON it.product_ID=p.id
JOIN basket_quotes bq ON bq.id=p.basket_quote_id;   -- 285 lines, 182 quotes, 2025-07-16 → 2026-09-28
```

For each row: installation type found and `surcharges.length >= 1 && === sqlCount` → resolves.
Result: **285 resolve, 0 false alarms, 0 on a different type, 23 on a product that no longer offers
installation** (all 23 are the 1.8m kit; only quote 3909's 2 lines are still open and valid).

**Q7 — surcharge counts and product log history (Mongo)**

```js
db.installationtypes.find({},{surcharges:1}).forEach(t => /* count by surcharges.length */)
// {0:1, 6:2, 7:7, 8:32}; the 0-surcharge type "Unknown" is used by 0 published products
db.products.countDocuments({status:'PUBLISHED', installationType:{$nin:[null,'']}})   // 658
db.productlogs.find({'current._id':ObjectId('5f16f3be8b49590d02dea1f6')})            // 2 rows, 2025-11-25
```

**Q8 — every stored installation row on dev vs Mongo surcharge count (MySQL → Mongo)**

```sql
SELECT CAST(it.mongo_id AS BINARY) type_id,
       (SELECT COUNT(*) FROM installation_surcharges s WHERE s.installation_type_ID=it.id) n_sql,
       COUNT(*) n_lines, MAX(it.created_at)
FROM installation_types it GROUP BY 1,2;   -- 52 groups, 7,873 rows
```

For each group: type found and `surcharges.length >= 1 && === n_sql` → resolves. Result: 7,847
resolve, 26 fail (listed in §7, Fix 3). The same surcharge-count distribution was read from the
local production dump (`cdn` db, root pair via `docker exec firstfence-mongo`).

**Blast radius of the shared "Palisade Fencing" type (MySQL)**

```sql
SELECT COUNT(*), SUM(p.basket_quote_id IS NOT NULL),
       SUM(bq.purchase_completed=0 AND bq.created_at >= NOW() - INTERVAL 5 DAY),
       SUM(c.id IS NOT NULL AND c.completed=0)
FROM installation_types it JOIN products p ON p.id=it.product_ID
LEFT JOIN basket_quotes bq ON bq.id=p.basket_quote_id LEFT JOIN carts c ON c.id=p.cart_ID
WHERE it.mongo_id='62ecec379f11e1c565a59c64';   -- 1726 / 59 / 2 / 1402
```

---

## Appendix B — the files that matter

| Repo | File | Why |
|---|---|---|
| `website-api` | `app/Utils/reorder-changes.js:179-285` | `evaluateLine` — order of checks, all facets |
| `website-api` | `app/Services/ReorderValidationService.js:30-61,147-169` | Loads docs; option and installation lookups (Fix 1 goes here) |
| `website-api` | `app/Utils/product-helpers.js:77-107` | `getAllVariantOptions` |
| `website-api` | `app/Utils/installation.js:13-36,45-186,195-206` | Save with `> 1`; `getInstallationType`; count check; `isInstallationAvailable` |
| `website-api` | `app/Controllers/Http/Portal/CustomerQuoteController.js:325-384,400` | `validate`, `convert-to-order` (Fix 2 goes in the second) |
| `website-api` | `app/Controllers/Http/BasketQuoteController.js:1506-1589,1707,1742-1752,1860` | Copies quote lines + installation; no lines in the body; installation price |
| `website-api` | `app/Utils/quote-validity.js:41` | Basket quotes live 5 days |
| `admin-website-v2` | `src/pages/content/product/main-fields/main-fields.jsx:43-50`, `normalize.js:114` | "Installation Not Available" → `null` |
| `cdn-graphql-v2` | `src/resolvers/product-variant.js:173-190`, `installation-type.js:105`, `surcharge.js:72`, `product.js:646` | All deletes are hard deletes |
| `ff-uk-mobile` | `src/features/quotes/screens/PayForQuoteScreen.tsx:60-92` | Quote flow reads only `valid` |
| `ff-uk-mobile` | `src/features/basket/components/BasketUpdatedSheet.tsx:168-212` | Change messages (used by reorder only) |
