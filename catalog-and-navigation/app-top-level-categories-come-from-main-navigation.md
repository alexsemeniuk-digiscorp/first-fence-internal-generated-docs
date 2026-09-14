# App "Shop by category" list does not change when you edit categories — it is built from Main Navigation

QA changed categories in the admin panel. QA reported adding one
(`https://admin.dev.firstfence.co.uk/content/categories/64de0601a9a4ba05c7ac1d57`) and archiving an
existing one. The mobile app's **Shop by category** list stayed the same: "Roadside Fencing & …"
(184 items) and "Home & Garden" (107 items). But a new subcategory, **"Bambura Test"**, did appear as
a chip inside Home & Garden. The question was: *am I missing something?*

**Data provenance:** all figures measured read-only on **2026-09-14** against the **dev** content
database (`cdn-test` @ `ec2-34-252-129-247.eu-west-1.compute.amazonaws.com`) and the **dev** GraphQL
API (`https://cdn.dev.firstfence.co.uk/graphql`, read-only queries only). Code read at
`ff-uk-mobile` **`0566c88`** (`main`; `origin/dev` at `0ceffaf` checked where it differs),
`cdn-graphql-v2` **`c7cc103`** (`master`; `dev-docker` checked where it differs), `admin-website-v2`
**`c4287e6`** (`main`), `gatsby-website` **`22fe4a6ca`** (`master`). No writes of any kind were made.
Production was not touched.

Screenshot times: the phone shows 15:40 and 15:45. I assume the phone is on Kyiv summer time
(EEST, UTC+3, the same as the machine the files were saved on), which is **12:40 and 12:45 UTC**.

---

## TL;DR

**Yes, one thing is missing: the top-level list in the app is not edited on the category page. It is
edited in Admin → Others → Main Navigation (`/content/main-navigation`).** The app reads the
children of the Main Navigation item "Fence By Category" and shows one card per child. A category's
own status, title or "newness" is never checked.

1. **Adding a category** does not add a card. Nothing creates a Main Navigation item for it.
2. **Archiving a category** does not remove its card. The app never reads the category's status, and
   the API returns archived categories too.
3. **The subcategory chip worked** because chips use a different source: the parent category's
   *Subcategories* tab. That is exactly the screen QA was editing.
4. **After a Main Navigation change, restart the app.** The card list is loaded once and kept for
   the whole app session. Leaving the screen and coming back does not reload it (§5).
5. **The card titles come from Main Navigation, not from the category.** That is why the app says
   "Home **&** Garden" while the category is called "Home **and** Garden". You can use this to check
   which list you are looking at.

There are two lists in admin that are both called **"Fence By Category"**, and today they hold the
same 7 entries. This is the most likely reason it looked like the category page should work (§3).

---

## 1. Where each part of the screen comes from

| What you see in the app | Where it is edited in admin | Source in the data |
|---|---|---|
| The list of top-level cards | **Others → Main Navigation**, children of "Fence By Category" | `mainnavigations` collection |
| Card **title** ("Home & Garden") | Main Navigation → node → **Name** | `mainnavigations.name` |
| Card **subtitle** ("Garden and boundary fencing") | Product Categories → category → **Short Description (mobile)** | `categories.shortDescription` (falls back to `metaDescription`) |
| Card **image** | Product Categories → category → **Images** tab | `categories.imageCollection` |
| Card **"107 items"** | Product Categories → category → **Products** tab (only products that are visible and published count) | `categories.products` array |
| Card **order** | Main Navigation → up/down arrows | `mainnavigations.sortOrder` |
| Subcategory **chips** ("Bambura Test") | Product Categories → parent category → **Subcategories** tab | `categories.subcategories` array |

So a category edit **can** change a card's subtitle, image and item count. It **cannot** add, remove,
rename or reorder a card.

### The code

`ff-uk-mobile/src/features/catalog/services/catalogApi.ts:164` —
`const CATEGORY_NAV_ROOT_LINK = '/fence-by-category';`

`ff-uk-mobile/src/features/catalog/services/catalogApi.ts:462-468`:

```ts
const root =
  nav.find((n) => !n.parentId && n.link === CATEGORY_NAV_ROOT_LINK) ??
  nav.find((n) => !n.parentId);
const children = nav
  .filter((n) => root && n.parentId === root.id && n.name && n.link)
  .sort((a, b) => (a.sortOrder ?? 0) - (b.sortOrder ?? 0));
```

Then for each child it loads the category **by path** and builds the card
(`catalogApi.ts:492-499`):

```ts
id: cat?.id ?? child.id,
title: child.name as string,
subtitle: cat?.shortDescription ?? cat?.metaDescription ?? '',
path: child.link as string,
imageUrl: cat ? mainImageUrl(cat) : null,
itemCount: counts?.pageInfo?.totalCount ?? null,
```

The category fragment (`catalogApi.ts:178-191`) does not even request `status`. On the server,
`allCategories` is a plain `models.Category.find(input)` with no status filter
(`cdn-graphql-v2/src/resolvers/category.js:9-10`).

The chips use a different query and filter only **DRAFT** (`catalogApi.ts:542`):

```ts
.filter((sub) => sub?.title && sub.path && sub.status !== 'DRAFT')
```

This screen has been built on Main Navigation since it was first written: `8bf8304`
(2026-07-15, "[FIR-9] Browse Categories"). It never read the categories list directly.

---

## 2. What the dev data looks like now

### Main Navigation — the list the app shows

Root item `694021a078aa45a3eb588f03`, name "Fence By Category", link `/fence-by-category`, created
2025-12-15. Its children, measured in Mongo and again through the live dev GraphQL API:

| sortOrder | Name (= card title) | Link | Category found at that path | Status | Sub-nodes |
|---|---|---|---|---|---|
| 0 | Security Fencing | `/security-fencing` | Security Fencing & Gates | PUBLISHED | 7 |
| 1 | Temporary Fencing | `/temporary-fencing` | Temporary Fencing, Hoarding and Crowd Control | PUBLISHED | 4 |
| 2 | Crash Barriers & Impact Protection | `/crash-and-impact-protection` | Crash and Impact Protection Barriers | PUBLISHED | 5 |
| 3 | Civils & Construction | `/civils-and-construction` | Civils & Construction | PUBLISHED | 7 |
| 4 | Agriculture | `/agriculture` | Agriculture | PUBLISHED | 5 |
| 5 | **Roadside Fencing & Barriers** | `/roadside-barriers` | Roadside Barriers and Fencing | PUBLISHED | 6 |
| 6 | **Home & Garden** | `/home-and-garden` | Home and Garden | PUBLISHED | 5 |

The app shows all 7. QA's screenshot is scrolled to the bottom of the list, so only the last two
are in view. The "Shop by category" header and the search bar stay fixed above the scrolling list
(`CategoriesScreen.tsx:20-31`).

The live dev API returns **184** products for `/roadside-barriers` and **107** for
`/home-and-garden` — exactly the numbers in the screenshot. So the app build QA used is on the dev
API and was showing current data.

No Main Navigation node was created today (newest node: 2026-05-19). Nav documents have no
`created`/`updated` fields, so an **edit** to an existing node cannot be dated.

### Categories edited today

| Time (UTC) | Category | Path | Status now | Subcategories now |
|---|---|---|---|---|
| 12:38:19 | **Fence By Category** | `/fence-by-category` | PUBLISHED | 7 |
| *12:40* | *screenshot 1* | | | |
| *12:45* | *screenshot 2 — "Bambura Test" chip visible* | | | |
| 12:49:15 | **Home and Garden** (QA's link) | `/home-and-garden` | PUBLISHED | 5 |
| 13:07:25 | Railway Sleepers | `/railway-sleepers` | PUBLISHED | 3 |
| 13:08:02 | Green Treated Railway Sleepers | `/green-treated-railway-sleepers` | PUBLISHED | 0 |

What this shows, and what it does not:

- **QA's link is not a new category.** `64de0601a9a4ba05c7ac1d57` is "Home and Garden", created
  **2023-08-17**. It is already the 7th card.
- **No category was created today.** The newest category id dates from 2026-06-19. No new image
  collection was created today either — every new category gets one.
- **No category is archived today.** The most recent change to an ARCHIVED category was on
  2025-12-19.
- **No category called "Bambura Test" exists now**, under any status. Home and Garden has 5
  subcategories now. In screenshot 2 it had a 6th chip, "Bambura Test", after "Ready Made Aluminum
  Driveway & Garden Gates". The Home and Garden save at 12:49:15 is four minutes after the
  screenshot.

So the archive and the "Bambura Test" subcategory were most likely undone after the screenshots.
I cannot prove this. Category change logging has been switched off since 2025-12-17 (see §8), so
there is no record of the edits themselves. **QA: if you remember which category you archived,
please say — I could not find it.**

---

## 3. Why it looked like the category page should work — two lists with the same name

| | Category **"Fence By Category"** | Main Navigation item **"Fence By Category"** |
|---|---|---|
| Admin screen | Products → Product Categories → `/content/categories/69a8591f87a796f378d3bf17` → Subcategories tab | Others → Main Navigation → `/content/main-navigation` |
| Created | 2026-03-04 | 2025-12-15 |
| Entries | 7 subcategories | 7 child nodes |
| Used by the **app** | **No** | **Yes** — the "Shop by category" cards |
| Used by the **website** | The `/fence-by-category` category page | The header menu |
| Edited today | Yes, 12:38 UTC | No new nodes |

Today both lists point at the same 7 paths in the same order. Only the names differ:

| Category subcategory title | Main Navigation name (what the app shows) |
|---|---|
| Security Fencing & Gates | Security Fencing |
| Temporary Fencing, Hoarding and Crowd Control | Temporary Fencing |
| Crash and Impact Protection Barriers | Crash Barriers & Impact Protection |
| Civils & Construction | Civils & Construction |
| Agriculture | Agriculture |
| Roadside Barriers and Fencing | **Roadside Fencing & Barriers** |
| Home and Garden | **Home & Garden** |

Someone keeps these two lists the same by hand. Nothing in the code links them. Editing one never
changes the other.

---

## 4. What QA should do

### To add a card

1. Open **Admin → Others → Main Navigation** (`/content/main-navigation`).
2. Find the root node **Fence By Category**. Click **Add** on it.
3. In the new node, set **Name** (this is the card title) and **Link** — the category's **Path**
   exactly as it is on the category page, for example `/home-and-garden`.
4. Use the up/down arrows to put it in the right place.
5. Make sure the category itself is **PUBLISHED**, has products, a Short Description (mobile) and an
   image — otherwise the card is empty (§6).
6. **Kill and restart the app** (§5).

### To remove a card

Removing is harder, and it has side effects. Read §6 first.

- The **Remove** button only appears on a node with **no children**
  (`admin-website-v2/src/pages/content/main-navigation-page/navigation-list/navigation-node.jsx:83`).
  Every one of the 7 card nodes has 4–7 children. To remove a card you must first remove all its
  children.
- **Those children are the website's header drop-down menu.** Removing them removes that menu
  section from the website too (after the next website build).

**Safer for a test:** do not remove anything. Open the existing node and change its **Name** and
**Link** to the new category. The children stay, and you can change it back later.

### To replace a category with a new one (what QA was trying to do)

1. Create and set up the new category (PUBLISHED, products, image, short description).
2. In **Main Navigation**, edit the old card's node: set **Link** to the new category's path, and
   change **Name** if needed.
3. Optionally, in the **"Fence By Category" category → Subcategories tab**, swap the entry too, so
   the website's `/fence-by-category` page matches.
4. Now archive the old category. Archiving it first does nothing visible in the app.
5. Restart the app.

---

## 5. Why a restart is needed — the app never reloads the card list

| Screen | When it reloads | What QA must do after an admin change |
|---|---|---|
| Top-level cards (Categories tab, Home "Popular categories", Search "Shop by category") | **Never while the app is running** on `main` / `stage` builds | **Kill and restart the app** |
| Same, on a build from `origin/dev` | When you pull down to refresh on the **Home** tab | Pull to refresh on Home, or restart |
| Subcategory chips and product counts inside a category | When you open the category screen again, **more than 60 seconds** after leaving it | Go back, wait a minute, open again — or restart |

Why:

- The API client sets no cache options (`ff-uk-mobile/src/store/api/graphqlApi.ts:40-45`), so the
  RTK Query defaults apply: data is kept 60 seconds after the last screen using it closes, and it is
  not reloaded on focus or on app resume.
- The card list is used by `HomeScreen.tsx:66`, `CategoriesScreen.tsx:15` and `SearchScreen.tsx:45`.
  Home is the first tab and is never closed (`src/navigation/MainTabs.tsx:40`), so the card list
  always has a screen using it. It is never dropped and never reloaded.
- Nothing ever marks the `'Category'` or `'Navigation'` data as stale.
- Only the Retry button on the error state reloads it (`CategoriesScreen.tsx:45`).
- A cold start always loads fresh data — nothing is saved to the phone.
- The **category screen** (with the chips) is closed when you go back, so it does reload. That is
  why the "Bambura Test" chip appeared and the cards did not — **on top of** the real reason in §1.
- Home pull-to-refresh exists only on `origin/dev`: added in `651facd` (2026-09-13); it refetches
  the stats and the card list. It is not on `main`.

**Important:** for QA's actual change — category added or archived, Main Navigation untouched — no
refresh and no restart would ever change the cards.

### A doc that is now wrong

`ff-uk-mobile/docs/ARCHITECTURE.md:34` and `ff-uk-mobile/AGENTS.md:19` both say admin changes to the
catalogue appear in the app **"immediately"**. That is true only for screens that open fresh. The
top-level card list, the Home "N Categories Live" count and the Search category list stay as they
were until the app restarts.

---

## 6. Traps

| # | Trap | Detail |
|---|---|---|
| 1 | **An archived category still shows as a card** | Tested on the dev API: `allCategories(path: "/impact-bollards")` returns the ARCHIVED "Bollards" category, and the product count for that path is **27**. A card linked to it would look completely normal. Today none of the nav links point at a non-published category (checked all of them). |
| 2 | **A wrong Link gives an empty card, not an error** | Link is free text (placeholder `'/test-page'`, `update-form.jsx:39-43`). There is no category picker and no check. Tested a path that does not exist: the API returns an empty list and a count of **0**. The app shows the card with the nav name, no subtitle, no image, and "0 items". |
| 3 | **Main Navigation is shared with the website header** | `gatsby-website/src/layout/header/navbar/navbar.js` reads the same `allMainNavigation`. Removing or renaming a node changes the website menu too, after the next website build (which is manual: `gatsby-website/.gitlab-ci.yml:37`). |
| 4 | **"For Testing?" hides a node on the website, not in the app** | The website drops `forTesting` nodes on live (`navbar.js:55-57`). The app does not even request the field (`catalogApi.ts:166-176`). So there is no way to hide a card in the app only. Today 1 node has it set ("Highway Post and Rail Fencing", a sub-node under Roadside — not a card). |
| 5 | **Archived and TEST subcategories still show as chips** | The app hides only DRAFT. The website shows only PUBLISHED (TEST only on its test build; `gatsby-website/src/templates/category/category.js:58-64`). On dev, published parents link to **9 ARCHIVED** and **10 TEST** subcategories that the app shows and the website hides — for example "Verge Posts" (ARCHIVED) under "Roadside Protection". Archiving a subcategory is not enough to hide its chip in the app; remove it from the parent's Subcategories tab. |
| 6 | **The item count ignores subcategories** | "107 items" counts only products listed directly on the category's Products tab, filtered to visible and published. Products that are only in a subcategory are not counted (`cdn-graphql-v2` branch `dev-docker`, `src/utils/category-product-connection.js:29-41`). Home and Garden has 132 product ids; 107 of them are visible and published. |
| 7 | **"Remove" on a category is a hard delete** | The trash icon on the category page calls `removeCategory`, which deletes the document (`cdn-graphql-v2/src/resolvers/category.js:99`). Any Main Navigation node or Subcategories entry pointing at it is left behind with nothing to show. Archive instead. |
| 8 | **Two categories with the same path** | `path` is not unique (`cdn-graphql-v2/src/models/category.js:7`). The app takes the first result. Today there are **0** duplicate paths on dev — but do not give a new category the old one's path while the old one still exists. |

---

## 7. Is this just dev?

**a. Can a deploy or pipeline cause this on production?**
No. Categories and Main Navigation are content, edited by hand in admin. No migration, seeder or CI
job writes either. The behaviour is in the app and API code, so it is the same on every environment.

**b. Can it happen again?**
Yes, every time. It is how the system is designed today, not a bug that was fixed. Anyone who edits
a category and expects the app's top-level list to follow will see the same thing.

**c. Is production already in this state?**
Not checked — production was not touched. For the app it does not matter yet: the app's production
GraphQL URL is still a placeholder, `cdn.example.invalid` (`ff-uk-mobile/app.config.ts:41`), and
the `development`, `dev` and `stage` build profiles all use the dev API (`eas.json:12,19,29`); the `production` profile (`eas.json:43`) points at that placeholder.

**One real production risk — for the backend team, not QA.** The item count on every card uses the
query `allCategoryProductsSearch`. That query exists only on the `cdn-graphql-v2` branches
`dev-docker` and `feature/paginated-category-product-list-query` (`e571f78`, 2026-07-16). It is
**not on `master`** (local refs last fetched 2026-09-03). `master` is the branch that deploys to
production (`cdn-graphql-v2/.gitlab-ci.yml:7-9`, `resource_group: production`). The app sends all
cards in one GraphQL document (`catalogApi.ts:470-481`), so on a CDN without that query the whole
document fails validation, and the Categories tab shows "Couldn't load categories". Check with:

```bash
cd cdn-graphql-v2 && git fetch origin && git branch -r --contains e571f78
```

---

## 8. How this fits with existing docs

| Doc | Status |
|---|---|
| `first-fence-internal-generated-docs/` | Nothing covered categories or Main Navigation before this doc. |
| `website-api/.generated_docs/FirstFence-Meetings.md:176-200` (Q8) | **Still correct.** It says the website navigation is "one central list of menu items" edited in admin, that the app "can read the exact same list", and recommends the Browse screen show "the main product categories only". That is what was built. |
| `ff-uk-mobile/docs/ARCHITECTURE.md:34`, `ff-uk-mobile/AGENTS.md:19` | **Needs correcting.** "Admin changes … appear in the app immediately" is not true for the top-level card list (§5). |
| `cdn-graphql-v2/.generated_docs/paginated-category-products-strategy.md` | **Still correct.** It says status and visibility come in through `filter` and nothing is special-cased, which matches the count behaviour in §6. |

**Category audit log.** `categorylogs` has 758 entries, from 2025-11-21 to **2025-12-17 15:43**.
It stops there because the `CategoryLog` writes in `cdn-graphql-v2/src/resolvers/category.js:40,71,89`
were commented out in `d66790a` (2025-12-17, "Remove history logs to save space and time"). That is
why QA's edits today cannot be reconstructed.

---

## 9. What I did not verify

- **Which category QA archived, and what "Bambura Test" was.** Neither exists in that state now, and
  there is no change log. §2 is an inference from `updated` timestamps.
- **The phone's time zone.** The UTC times of the screenshots assume EEST.
- **Which app build QA used** (`main`, `stage` or `dev`). It affects only whether Home
  pull-to-refresh exists (§5).
- **Edits to existing Main Navigation nodes today.** Nav documents have no timestamps. I only proved
  that no node was *created* today, and that the current tree matches the screenshot.
- **The oplog.** I tried to read the dev Mongo oplog to see today's writes; the `firstfence` user is
  not allowed to read `local.oplog.rs`.
- **The website.** I read its navbar and category template code but did not open the dev website or
  check its build date.
- **Production**, in any form, including whether `allCategoryProductsSearch` is already on the
  production CDN.
- I did not do an end-to-end test (edit nav → restart app → see change). That needs a write to dev,
  which this investigation does not do.

---

## Appendix A — queries used

**Dev Mongo** (`cdn-test`), through the local container:

```bash
docker exec -i firstfence-mongo mongosh --quiet \
  "mongodb://firstfence:<pass>@ec2-34-252-129-247.eu-west-1.compute.amazonaws.com:27017/cdn-test?authSource=cdn-test" \
  --eval "$(cat script.js)"
```

QA's category, and categories edited today:

```js
const C = db.getSiblingDB('cdn-test').categories;
C.findOne({_id: ObjectId("64de0601a9a4ba05c7ac1d57")});
C.find({updated: {$gte: ISODate("2026-09-14T00:00:00Z")}},
       {title:1, path:1, status:1, updated:1, subcategories:1}).sort({updated:1});
C.find({}, {title:1, created:1}).sort({_id:-1}).limit(5);        // newest by id
C.find({status:"ARCHIVED"}, {title:1, updated:1}).sort({updated:-1}).limit(8);
C.find({title: {$regex: "bambura|test", $options: "i"}}, {title:1, status:1});
```

Main Navigation tree:

```js
const N = db.getSiblingDB('cdn-test').mainnavigations;
N.find({parentId: null}).sort({sortOrder: 1});
N.find({parentId: "694021a078aa45a3eb588f03"}).sort({sortOrder: 1});
N.countDocuments({forTesting: true});
```

Subcategory links by child status:

```js
const byId = {};
C.find({}, {status:1, title:1}).forEach(c => byId[String(c._id)] = c);
const tally = {};
C.find({status:"PUBLISHED"}, {subcategories:1}).forEach(p =>
  (p.subcategories || []).forEach(id => {
    const st = byId[id] ? byId[id].status : "MISSING";
    tally[st] = (tally[st] || 0) + 1;
  }));
tally;   // { PUBLISHED: 506, ARCHIVED: 9, TEST: 10, DRAFT: 2 }
```

Duplicate paths and audit log range:

```js
C.aggregate([{$group:{_id:"$path", n:{$sum:1}}}, {$match:{n:{$gt:1}}}]);   // []
const L = db.getSiblingDB('cdn-test').categorylogs;
L.countDocuments({});                         // 758
L.find({}).sort({_id:-1}).limit(1);           // last: 2025-12-17 15:43
```

**Dev GraphQL** (`https://cdn.dev.firstfence.co.uk/graphql`), read-only — the same shape the app
sends:

```graphql
query {
  nav: allMainNavigation { id parentId name link sortOrder }
  rb: allCategoryProductsSearch(input: { path: "/roadside-barriers",
        filter: { visible: true, status: PUBLISHED }, limit: 1 }) { pageInfo { totalCount } }   # 184
  hg: allCategoryProductsSearch(input: { path: "/home-and-garden",
        filter: { visible: true, status: PUBLISHED }, limit: 1 }) { pageInfo { totalCount } }   # 107
  hgc: allCategories(input: { path: "/home-and-garden" }) {
    title status shortDescription subcategories { title status path } }
}
```

Archived and missing paths:

```graphql
query {
  arch: allCategories(input: { path: "/impact-bollards" }) { title status }      # ARCHIVED, returned
  archCount: allCategoryProductsSearch(input: { path: "/impact-bollards",
    filter: { visible: true, status: PUBLISHED }, limit: 1 }) { pageInfo { totalCount } }   # 27
  bogus: allCategories(input: { path: "/no-such-category-xyz" }) { id }          # []
  bogusCount: allCategoryProductsSearch(input: { path: "/no-such-category-xyz",
    filter: { visible: true, status: PUBLISHED }, limit: 1 }) { pageInfo { totalCount } }   # 0, no error
}
```

## Appendix B — the files that matter

| Repo | File:line | What it does |
|---|---|---|
| `ff-uk-mobile` | `src/features/catalog/services/catalogApi.ts:164` | Root link `/fence-by-category` |
| `ff-uk-mobile` | `src/features/catalog/services/catalogApi.ts:457-504` | Builds cards from Main Navigation |
| `ff-uk-mobile` | `src/features/catalog/services/catalogApi.ts:538-549` | Subcategory chips, hides DRAFT only |
| `ff-uk-mobile` | `src/store/api/graphqlApi.ts:40-45` | No cache options — RTK defaults |
| `ff-uk-mobile` | `src/features/catalog/screens/HomeScreen.tsx:66` | Keeps the card list loaded forever |
| `ff-uk-mobile` | `app.config.ts:30-41` | Dev URL for test/staging; production is a placeholder |
| `cdn-graphql-v2` | `src/resolvers/main-navigation.js:5-6` | `MainNavigation.find(input)`, no filter |
| `cdn-graphql-v2` | `src/resolvers/category.js:9-10` | `allCategories`, no status filter |
| `cdn-graphql-v2` | `src/resolvers/category.js:99` | `removeCategory` hard delete |
| `cdn-graphql-v2` | `src/resolvers/category.js:40,71,89` | Category log writes, commented out 2025-12-17 |
| `admin-website-v2` | `src/layout/routes/data/content.js:253-258` | Route `/content/main-navigation` |
| `admin-website-v2` | `src/layout/routes/links/content.js:238-251` | Menu: Others → Main Navigation |
| `admin-website-v2` | `src/pages/content/main-navigation-page/update-form/update-form.jsx:30-85` | Name, Link (free text), colours, alignment, For Testing |
| `admin-website-v2` | `src/pages/content/main-navigation-page/navigation-list/navigation-node.jsx:83` | Remove only when no children |
| `admin-website-v2` | `src/pages/content/category/edit-category.jsx:235-283` | Category fields — no "top level" or "show in menu" |
| `gatsby-website` | `src/layout/header/navbar/navbar.js:55-58` | Website header uses the same nav, hides For Testing |
