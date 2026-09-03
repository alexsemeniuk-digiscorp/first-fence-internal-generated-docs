# New price tier added in admin, but not shown on the website or in the mobile app

QA added a price tier (**buy 2000+, £10 each**) to one product in the admin panel. The tier did not
appear on the dev website, and it did not appear in the mobile app either. This document explains why.

Product under test:

| Field | Value |
|---|---|
| Title | PaliGuard® HD Fencing Pale to Suit 3.0m High System |
| SKU | `FF-WEB-0438` |
| Mongo id | `698f18bf2d4a743dbae14c8e` |
| Path | `/paliguard-fencing-pale-to-suit-3-0m-high-system` |
| Dev page | https://dev.firstfence.co.uk/paliguard-fencing-pale-to-suit-3-0m-high-system/ |

Everything below was measured on **3 September 2026** against the **dev** content database
(`cdn-test` on `ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`), the **dev** GraphQL API
(`cdn.dev.firstfence.co.uk`) and the **dev** website (`dev.firstfence.co.uk`). No code was changed.

---

## TL;DR

**QA did nothing wrong. The tier was saved correctly. There are three separate problems, and they
are not the same problem.**

1. **The website is a static site.** It does not read the database when a visitor opens a page. It
   reads a copy that was frozen at build time. The dev website was last built on
   **8 July 2026** — almost two months before QA made the change. So *no* content change made in
   admin since July is visible on dev. **A build must be triggered by hand** from
   Admin → Website Settings → Test Site → **Trigger build**. Nothing does this automatically.

2. **The mobile app is not static and the data is already there.** The dev API returns all six
   tiers, including 2000 @ £10, right now. So the app will show the tier **if it is pointed at the
   dev API**. The most likely cause is that the app build QA used was pointed somewhere else — the
   default app environment is `local` (`localhost:4000`), and the `.env.local` file in the repo
   points at a developer's own machine (`192.168.0.177:4201`), which runs against a **copy of the
   production database** where this tier does not exist.

3. **Even after a rebuild, this tier can never be used.** The product has
   `orderLimitQuantity = 2`, which means a customer may order at most **2** of them. A tier that
   starts at 2000 can never be reached. Both the mobile app and the basket API silently push the
   quantity back down to 2. This is a real data problem on this product and should be fixed
   separately.

**What to do next:** trigger a dev site build, check the app is pointed at
`https://cdn.dev.firstfence.co.uk/graphql`, and remove or raise the order limit of 2 on this product.

---

## 1. Was the tier actually saved? Yes.

The admin panel wrote the change correctly. In the dev database the product now has six tiers:

| Tier id | Buy this many or more | Price each |
|---|---|---|
| 145 | 50 | £14.93 |
| 146 | 150 | £13.65 |
| 147 | 300 | £13.22 |
| 148 | 900 | £12.79 |
| 149 | 1800 | £12.37 |
| **150** | **2000** | **£10.00** |

The product record was updated at **2026-09-03 08:43:57 UTC**. The product is `PUBLISHED` and
`visible: true`. The numbers are stored as real numbers, not text.

For comparison, the production database (`cdn`, same server) still has only the first five tiers.
That is expected — QA edited dev, not production.

So the admin panel, the save, and the database are all fine. The problem is further down the line.

---

## 2. Does the API return the tier? Yes.

The dev GraphQL API returns all six tiers. We checked this using the **exact query the mobile app
sends** (`ff-uk-mobile/src/features/product/services/productDetail.graphql`, operation
`ProductDetail`), not a simplified one:

```
priceTiers: [ 50/£14.93, 150/£13.65, 300/£13.22, 900/£12.79, 1800/£12.37, 2000/£10.00 ]
```

The API also sends `cache-control: no-store` and there is no CDN in front of it, so nobody is
serving an old copy of this response.

**This means the backend is not the problem.**

---

## 3. Why the website does not show it — the site is a frozen copy

This is the main cause, and it explains the website completely.

The website (`gatsby-website`) is a **static site**. Gatsby downloads all products from the API
**once, at build time**, and writes them into static files. A visitor never touches the database.
Until somebody builds the site again, the visitor sees whatever was true on the day of the build.

### The evidence

The dev site stores each page's data in a file. For this product:

```
https://dev.firstfence.co.uk/page-data/paliguard-fencing-pale-to-suit-3-0m-high-system/page-data.json
```

That file contains only **five** tiers — the 2000 @ £10 tier is missing. And its timestamp is:

```
last-modified: Wed, 08 Jul 2026 13:32:23 GMT
```

We checked other pages too. The home page and the site-wide data file carry the same date:

| File | Last built |
|---|---|
| this product's page data | 8 July 2026, 13:32:23 |
| 1.8m sibling product page data | 8 July 2026, 13:32:23 |
| home page data | 8 July 2026, 13:32:28 |
| site-wide `app-data.json` | 8 July 2026, 13:32:19 |

So the **whole dev website** was built once, on 8 July 2026, and has not been rebuilt since. Any
content change made in admin after that date is invisible on dev. This is not specific to price
tiers — it applies to titles, prices, images, everything.

The product page *does* render the multibuy block (the text "Multibuy Discounts Available" is in the
page, along with the five old prices £14.93, £13.65, £13.22, £12.79, £12.37). It is simply showing
July's data.

### Why no rebuild happened

The deploy pipeline (`gatsby-website/.gitlab-ci.yml`) only pulls the code onto the server. It ends
with this line, in the pipeline itself:

```
echo "Note: Gatsby build must be triggered manually from the admin site"
```

So there is **no automatic rebuild** — not on a content change, not on a schedule, not on deploy.
Somebody has to press a button.

### How to rebuild

In the admin panel: **Website Settings → Test Site → Trigger build**. There is a small arrow next to
the button with a **Clear cache** option.

Behind the scenes this sends a `POST` to `{VITE_WEBSOCKET_HOST_TEST}/trigger-build`
(`admin-website-v2/src/pages/website-settings/test-site/test-builds-page.jsx`). The live site has
its own equivalent page.

**Use "Clear cache" if a normal build does not pick the change up.** Gatsby keeps a per-product
cache keyed on the product's `updated` timestamp
(`gatsby-website/plugins/gatsby-source-firstfence/gatsby-node.js`). If the timestamp changed — and
here it did — a normal build is enough. Clearing the cache forces a full re-download.

---

## 4. Why the mobile app does not show it — most likely the wrong backend

The mobile app is **not** static. It asks the API every time the product page opens
(`graphqlApi.ts` even says: *"Read-only at runtime — no static build like the Gatsby site."*).

We checked the app code and found **nothing that would hide the tier**:

- The product detail query **does** ask for `priceTiers`.
- `buildProductBlocks.ts` adds the multibuy block whenever the product is buyable and has at least
  one tier. This product is `SIMPLE`, so it qualifies.
- `MultibuyBlock.tsx` only hides itself when the product is price-on-application, or has
  `ignoreBasePrice`, or has no tiers. None of those apply here.
- `multibuyButtons()` does not filter tiers by any limit. With six tiers it produces seven buttons:
  Buy 1+, 50+, 150+, 300+, 900+, 1800+, **2000+**.
- There is **no offline storage** (no `redux-persist`, no AsyncStorage cache) that could serve an
  old copy.

So a mobile app pointed at dev **will** show "Buy 2000+ / £10.00 ea".

### The likely cause: which server the app was talking to

The app picks its backend from `APP_ENV` (`ff-uk-mobile/app.config.ts`):

| `APP_ENV` | GraphQL URL |
|---|---|
| `local` (**the default**) | `http://localhost:4000/graphql` |
| `test` | `https://cdn.dev.firstfence.co.uk/graphql` |
| `staging` | `https://cdn.dev.firstfence.co.uk/graphql` |
| `production` | `https://cdn.example.invalid/graphql` (placeholder, not real yet) |

On top of that, `.env.local` in the repo overrides it to a **developer's own machine**:

```
EXPO_PUBLIC_GRAPHQL_URL=http://192.168.0.177:4201/graphql
```

That machine runs `cdn-graphql-v2` against a **local copy of the production database**, which has
only the five old tiers. If the app QA tested was started from the repo (Expo dev server) rather
than from a proper dev build, this is exactly what she would see: the product, but without the new
tier.

The EAS build profiles `development`, `dev` and `stage` all set `APP_ENV` correctly, so an app
installed from one of those **should** be fine.

### How to confirm in one minute

Ask QA which app she used, then check the app is talking to `cdn.dev.firstfence.co.uk`. Anyone can
verify the backend has the data with:

```bash
curl -s -X POST https://cdn.dev.firstfence.co.uk/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ product(id:\"698f18bf2d4a743dbae14c8e\"){ title priceTiers { amount price } } }"}'
```

If that command shows `2000 / 10` (it does today) and the app does not, the app is pointed at the
wrong server or needs a restart.

There is one much smaller possibility: the app keeps a query result in memory for a short time while
the screen stays open. Fully closing and reopening the app rules that out.

---

## 5. A separate real problem: this tier can never be used

This one is worth fixing regardless of the two causes above.

The product has:

```
orderLimitQuantity = 2
```

That means a customer may buy at most **2** of these pales. But the cheapest tier starts at **2000**.
The two settings contradict each other.

The limit is enforced in two places:

- **Mobile app** — `ff-uk-mobile/src/core/pricing/configurator.ts`: when the quantity is set above
  the limit, it is pushed back down to the limit. So tapping "Buy 2000+" would set the quantity to
  **2**, and the price would stay at £15.79. The button would look broken.
- **Basket API** — `website-api/app/Controllers/Http/CartController.js` and `ProductController.js`
  both do `Math.min(qty, orderLimitQuantity)`. So even a hand-made request cannot get 2000 into the
  basket.

The website passes `orderLimitQuantity` through to the basket but does not appear to cap the
quantity box on the page itself, so on the website the mismatch would only show up at basket stage.

### This limit looks accidental

- All five sibling products (1.5m, 1.8m, 2.0m, 2.1m, 2.4m PaliGuard pales) have
  `orderLimitQuantity = null`.
- The **production** copy of this same product also has `orderLimitQuantity = null`.
- Across the entire dev database, only **5 products** have an order limit set at all, and this is
  the **only one** where the limit is lower than its cheapest tier.

So the value of 2 exists only on dev, only on this product. It was most likely typed in during
testing.

**We cannot prove who set it or when.** Product change logging is switched off — the code that
writes a `ProductLog` is commented out in `cdn-graphql-v2/src/resolvers/product.js`, and the
`productlogs` collection has **0 entries** for this product. That is a gap worth raising on its own:
right now nothing records product edits on dev.

**Recommendation:** clear `orderLimitQuantity` on this product (or set it to something above 2000)
before re-testing the tier.

---

## 6. One smaller thing we noticed

In `gatsby-website/src/models/Product.js` the "lowest price" shown for a product is taken as the
**last** tier in the list:

```js
this.lowestPrice = product.priceTiers[product.priceTiers.length - 1].price
```

This assumes the tiers are always stored cheapest-last. It is true for this product today. But if
somebody ever adds a tier out of order in admin, the "from" price on category pages would be wrong.
Sorting by amount before taking the last item would make this safe. Not causing today's issue.

---

## 7. Summary table

| Where | Does it have the new tier? | Why |
|---|---|---|
| Admin panel | Yes | Saved correctly |
| Dev database (`cdn-test`) | **Yes** — 2000 @ £10 | Written 3 Sep 2026, 08:43 UTC |
| Dev GraphQL API | **Yes** | Live read, no caching |
| Dev website | **No** | Static build frozen on 8 July 2026 — needs a manual build |
| Mobile app (pointed at dev) | Should be **yes** | Nothing in the app hides it |
| Mobile app (pointed at local / a dev machine) | **No** | That backend uses a production data copy with 5 tiers |
| Production database (`cdn`) | No | Not changed — correct, QA edited dev |
| Can a customer actually buy at this tier? | **No** | `orderLimitQuantity = 2` caps the order at 2 items |

---

## 8. Actions

1. **Trigger a dev site build** — Admin → Website Settings → Test Site → Trigger build. Use
   *Clear cache* if a plain build does not show the tier. Then re-check the product page.
2. **Confirm which app build QA used**, and that it points at
   `https://cdn.dev.firstfence.co.uk/graphql`. Restart the app fully before re-testing.
3. **Remove `orderLimitQuantity = 2`** from this product, or raise it above 2000. Otherwise the
   tier will still look broken after the rebuild.
4. **Consider a follow-up ticket:** nobody is told that the website needs a manual rebuild after a
   content change. Every content edit on dev will look "lost" until someone presses the button.
   Either rebuild on a schedule, or show the last build date in the admin panel next to the content.
5. **Consider a follow-up ticket:** turn product change logging back on, so edits like this can be
   traced.

---

## Appendix — how to reproduce these checks

Dev database (password is in the commented-out "external test Mongo" block of `website-api/.env`):

```bash
docker exec -i firstfence-mongo mongosh --quiet \
  --host ec2-34-252-129-247.eu-west-1.compute.amazonaws.com --port 27017 \
  -u firstfence -p '<MONGO_DB_PASSWORD>' \
  --authenticationDatabase cdn-test cdn-test \
  --eval 'printjson(db.products.findOne(
    { path: "/paliguard-fencing-pale-to-suit-3-0m-high-system" },
    { title: 1, priceTiers: 1, orderLimitQuantity: 1, updated: 1 }))'
```

Dev API:

```bash
curl -s -X POST https://cdn.dev.firstfence.co.uk/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ product(id:\"698f18bf2d4a743dbae14c8e\"){ title orderLimitQuantity priceTiers { amount price } } }"}'
```

What the dev website is actually serving, and when it was built:

```bash
curl -s https://dev.firstfence.co.uk/page-data/paliguard-fencing-pale-to-suit-3-0m-high-system/page-data.json \
  | grep -o '"priceTiers":\[[^]]*\]'

curl -sI https://dev.firstfence.co.uk/page-data/app-data.json | grep -i last-modified
```

The last command is the quickest way to answer *"is the dev site stale?"* for any future report.
