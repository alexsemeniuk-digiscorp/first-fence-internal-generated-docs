# New price tier added in admin, but not shown on the website or in the mobile app

QA added a price tier (**buy 2000+, £10 each**) to one product in the admin panel. The tier did not
appear on the dev website, and it did not appear in the mobile app either. Later the same day the
tier **started showing in the mobile app on its own**, with no rebuild and no change on the server.
This document explains all of it.

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

2. **The mobile app was showing a cached copy.** The app is not static — it asks the API every
   time. The API has had the tier since 08:43. The app later started showing it with no rebuild and
   no change on the server, which points at the app's own memory. The app keeps query results for
   **60 seconds** after a screen stops using them, and — this is the important part — it **never
   refetches on its own**: not when a screen is re-opened, not when the app comes back from the
   background, not when the network reconnects. All three of those switches are off. So a product
   screen left sitting in the app can show the same old data for hours. Fully closing and reopening
   the app is what clears it. (A second possibility: the app build was pointed at a different
   backend — see section 4.)

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

## 4. Why the mobile app did not show it — the app's own memory

**Update:** the tier now shows in the app, with no rebuild, no deploy, and no change in the
database. We re-checked: the product record is still `updated: 2026-09-03 08:43:57 UTC` and all six
tier entries still carry that same timestamp, so nobody re-saved it. That tells us the cause was
never on the server.

The mobile app is **not** static. It asks the API every time the product screen opens
(`graphqlApi.ts` even says: *"Read-only at runtime — no static build like the Gatsby site."*).
Nothing in the app hides the tier — the query asks for `priceTiers`, the multibuy block is added
whenever a buyable product has at least one tier, and the button list is not filtered. With six
tiers the app draws seven buttons, ending in **BUY 2000+ / £10.00 ea**.

### There is no cache on the server. We checked all of it.

| Layer | Result |
|---|---|
| Apollo response cache (`responseCachePlugin`) | Not installed, not configured |
| `@cacheControl` / `maxAge` hints in the schema | None — zero matches across all 44 schema files |
| A `priceTiers` field resolver | **None** — served straight off the live database record |
| DataLoader, memoisation, module-level caches | None |
| Mongoose cache plugins (`cachegoose`, redis) | None — no cache library in `package.json` at all |
| Data loaded once at server boot | None |
| Proxy / CDN in front | Caddy, pass-through. No `age`, no `x-cache`, response times flat |
| Response headers | `cache-control: no-store`, sent by Apollo itself |

Apollo sets `no-store` because there are no cache hints anywhere, so even a CDN in front is told
not to store the response. The requests are also `POST`, which proxies do not cache anyway.

**So a tier written to the database is visible to the very next API call.** Confirmed by running the
app's own query against dev: all six tiers come back.

### The one cache that can do this is inside the app

The app uses RTK Query (Redux Toolkit 2.12.0). It keeps each query result in memory, and:

- results are kept for **60 seconds** after the last screen using them goes away — this is the
  library default and the code does not change it for the product query;
- **`refetchOnMountOrArgChange` is off** — re-opening a product screen serves the stored copy
  without asking the server;
- **`refetchOnFocus` is off** — bringing the app back from the background does not refresh anything;
- **`refetchOnReconnect` is off** — reconnecting to the network does not refresh anything;
- there is no polling and no background revalidation.

(`setupListeners` *is* called in `src/store/index.ts`, but it does nothing here: it only helps
screens that opted into focus/reconnect refetching, and none did.)

**The 60 seconds is not really 60 seconds.** The countdown only starts when the *last* screen using
that product stops using it. Several places open a product with `navigation.push` —
`ProductSwitchButton.tsx`, `RelatedProductsBlock.tsx`, `RecentlyViewedBlock.tsx` — which puts a new
product screen **on top of** the old one while the old one stays alive and still holds its copy. A
product left in the back stack therefore holds its original data **for as long as the app is
running**, and pressing back shows that original data again with no network call.

Backgrounding the app does **not** restart it, so the memory survives. Only a full close-and-reopen
(or a Metro reload in a dev build) clears it — and there is no saved-to-disk copy of API data
anywhere (no redux-persist, no AsyncStorage, no MMKV, no SQLite), so a real restart always refetches.

### The sequence that matches what QA saw

1. QA opened the product page in the app at some point in the session.
2. The tier was added in admin at 08:43. The API had it immediately.
3. The app never asked again — the screen was still alive, or she came back to it inside the
   60-second window, or she just backgrounded the app rather than closing it.
4. Later the app was fully closed and reopened (or the screen was left long enough to expire), the
   query ran again, and the tier appeared.

No cache was cleared and nothing was fixed — the app simply asked the server for the first time
since the change.

### The other possible cause: a different backend

This one is still worth ruling out, because it produces exactly the same symptom.

The app picks its backend from `APP_ENV` (`ff-uk-mobile/app.config.ts`):

| `APP_ENV` | GraphQL URL |
|---|---|
| `local` (**the default**) | `http://localhost:4000/graphql` |
| `test` | `https://cdn.dev.firstfence.co.uk/graphql` |
| `staging` | `https://cdn.dev.firstfence.co.uk/graphql` |
| `production` | `https://cdn.example.invalid/graphql` (placeholder, not real yet) |

`EXPO_PUBLIC_GRAPHQL_URL` in `.env.local` overrides all of that, and **`.env.local` is not in git**
— every machine has its own. So which server a build talks to depends on the machine it was built
on. A build pointed at a local or LAN copy of the API would read the **local production dump**,
which has only the five old tiers and `orderLimitQuantity = null`. We confirmed that dump directly.

The EAS profiles `development`, `dev` and `stage` all set `APP_ENV=test`, so an app installed from
one of those talks to dev and is fine.

The app has **no over-the-air updates** (`expo-updates` is not installed), so the app's code cannot
have changed by itself between the two attempts — only the data it fetched did.

### How to rule the endpoint out in one minute

```bash
curl -s -X POST https://cdn.dev.firstfence.co.uk/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ product(id:\"698f18bf2d4a743dbae14c8e\"){ title priceTiers { amount price } } }"}'
```

If that shows `2000 / 10` (it does) and an app does not, that app is either holding an old copy in
memory or pointed at a different server. **Always fully close and reopen the app before reporting a
content change as missing.**

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

## 6. Two smaller things we noticed

### A saved price that never expires

`ff-uk-mobile/src/store/slices/recentlyViewedSlice.ts` saves the last 8 viewed products **to disk**,
including a copy of `price`, `fromPrice`, `pricePerMeter` and `priceUnit`. That copy has **no expiry
and is never invalidated** — it is only wiped when a different user signs in. It survives app
restarts.

It does **not** store `priceTiers`, so it is not the cause of this report. But it does mean the
price shown on a card in the "Recently viewed" row can stay wrong indefinitely after a price change.
Worth a ticket of its own.

### A "lowest price" that assumes sorted tiers

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
| Mobile app, first attempt | **No** | The app was still holding the copy it fetched before the change |
| Mobile app, after a full restart | **Yes** | It asked the server again and got all six tiers |
| Mobile app pointed at a local / LAN backend | **No** | That backend uses a production data copy with 5 tiers |
| Production database (`cdn`) | No | Not changed — correct, QA edited dev |
| Can a customer actually buy at this tier? | **No** | `orderLimitQuantity = 2` caps the order at 2 items |

---

## 8. Actions

1. **Trigger a dev site build** — Admin → Website Settings → Test Site → Trigger build. Use
   *Clear cache* if a plain build does not show the tier. Then re-check the product page.
2. **Re-test the app after fully closing and reopening it.** Backgrounding is not enough — the app
   never refreshes by itself. Also confirm the build points at
   `https://cdn.dev.firstfence.co.uk/graphql` and not a local or LAN backend.
3. **Remove `orderLimitQuantity = 2`** from this product, or raise it above 2000. Otherwise the
   tier will still look broken after the rebuild.
4. **Consider a follow-up ticket:** nobody is told that the website needs a manual rebuild after a
   content change. Every content edit on dev will look "lost" until someone presses the button.
   Either rebuild on a schedule, or show the last build date in the admin panel next to the content.
5. **Consider a follow-up ticket:** turn product change logging back on, so edits like this can be
   traced.
6. **Consider a follow-up ticket:** the app never refreshes product data on its own. Turning on
   `refetchOnFocus` (refresh when the app returns to the foreground) and/or
   `refetchOnMountOrArgChange` for the product query would stop this class of "I changed it but the
   app still shows the old thing" report — for QA now, and for customers after a price change later.
7. **Consider a follow-up ticket:** the "Recently viewed" row stores prices on disk with no expiry
   (section 6).

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
