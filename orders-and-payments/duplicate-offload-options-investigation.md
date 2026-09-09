# Duplicate offload options in `/delivery` — where the second £30 / £89 pair came from

QA hit `POST /delivery` on dev and got **two `ASSISTED_OFFLOAD` rows (£30 and £43)**, **two
`FIRST_FENCE_OFFLOAD` rows (£89 and £108)** and **two `SELF_OFFLOAD` rows**, all rendered in the
mobile app checkout. The question was whether this is a staging-only artefact or a bug in the price
update logic, and whether the stale £30/£89 values are the dead `OFFLOAD_PRICES` constant that
[`delivery-types-and-options.md` §6](./delivery-types-and-options.md) describes.

**Data provenance:** all figures verified read-only against the **dev database**
(`website_test` @ `54.171.181.199`) on **2026-09-09**. Code read at `website-api` `0f386f2`
(branch `mobile-app`), plus `ff-uk-mobile`, `gatsby-website` and `admin-website-v2` at their current
tips. No writes of any kind were made.

---

## TL;DR

**It is a data problem on dev, not a bug in the price update logic — but it is not harmless, and
"it'll never happen on prod" is only true going forward.**

1. **The £30 / £89 QA sees are NOT `OFFLOAD_PRICES`.** §6 of the delivery doc is still correct:
   `OFFLOAD_PRICES` (`app/Utils/constants.js:69`) is declared once and referenced nowhere in `app/`
   or `start/` — dead code. The £30/£89 QA sees come from **real rows in `delivery_offloads`**. The
   values match by common ancestry, not by cause: both the constant and the duplicate rows are
   snapshots of the same pre-June-2026 pricing.

2. **`delivery_offloads` on dev holds 8 rows where it should hold 4.** Ids 5/6/7/10 are the real
   rows, live since 2020 and carrying all the tier configuration. **Ids 1/2/3/4 are copies** with
   the old prices and **no tier rows at all**.

3. **The price update logic never created them.** The admin edit form is a strict
   `PUT /delivery/offload/:id`; there is no create-on-save path anywhere. And it *couldn't* have —
   the table's `AUTO_INCREMENT` is **15**, so anything created through the admin UI would land at id
   ≥ 11, never at 1–4. Explicit low ids can only come from an INSERT that names them.

4. **They came from one manual `adonis seed` run on 2026-09-01.**
   `database/seeds/DeliveryOffloadSeeder.js` hard-codes ids 1–4 with exactly these prices, names,
   details, `sort_order` **and `created_at`/`updated_at` literals** — the dev rows match the seed
   file byte for byte. The run is recorded in the session log for that day; it is also why only
   this one table was hit (§4).

5. **The impact is real money, not just a duplicated dropdown.** Nothing in the API re-validates or
   re-prices the choice — `POST /order/create` stores whatever the client sends. **66 dev deliveries
   since 2 September point at the phantom rows**, and the duplicates bypass every business rule the
   tier configuration encodes:

   | Cart | Business rule (rows 5/6/7/10) | What the duplicates let through |
   |---|---|---|
   | over £1,000 | Assisted Offload **withdrawn** | offered at **£30** (id 3) |
   | over £1,000 | First Fence Offload → **£150** | also offered at **£89** (id 2) |
   | over £3,000 | Customer Offload **withdrawn** | still offered (id 1) |
   | any | Customer Offload (Mechanical) hidden below £3,000 | always offered (id 4) |

6. **Production: not affected by anything automatic, and it can no longer happen.** Neither GitLab
   pipeline runs `adonis seed` or `adonis migration:run` — the current one is `git reset --hard` +
   `pm2 restart`, the old one `docker build` + `ecs deploy`. Seeding is a manual command only. And
   since `30ee315` (2026-09-01) `seedIfEmpty` skips any table that already has rows, so even a
   manual run is now inert. **The one thing not verified is whether prod's table is already clean**
   — §5 gives the read-only query to settle it.

7. **Fix on dev is one statement:** `UPDATE delivery_offloads SET enabled = 0 WHERE id IN (1,2,3,4);`
   Do **not** `DELETE` — the FK is `ON DELETE SET NULL` and would silently orphan 66 delivery rows.

---

## 1. What QA is actually looking at

### 1.1 The eight rows

`delivery_offloads` on `website_test`, 2026-09-09 — all eight `enabled = 1`, `is_deleted = 0`:

| id | `type` | price | `updated_at` | tier rows in `delivery_offload_options` | verdict |
|---|---|---|---|---|---|
| **1** | `SELF_OFFLOAD` | — | 2023-03-31 | **none** | copy |
| **2** | `FIRST_FENCE_OFFLOAD` | **£89.00** | 2023-03-31 | **none** | copy |
| **3** | `ASSISTED_OFFLOAD` | **£30.00** | 2023-03-31 | **none** | copy |
| **4** | `CUSTOMER_OFFLOAD_MECH` | — | 2023-03-31 | **none** | copy |
| 5 | `SELF_OFFLOAD` | — | 2023-03-31 | id 18 | real |
| 6 | `FIRST_FENCE_OFFLOAD` | **£108.00** | **2026-06-02** | id 12 | real |
| 7 | `ASSISTED_OFFLOAD` | **£43.00** | **2026-06-02** | id 3 | real |
| 10 | `CUSTOMER_OFFLOAD_MECH` | — | 2023-03-31 | ids 16, 19 | real |

`AUTO_INCREMENT = 15`; ids 8, 9, 11–14 are gaps from offloads created and deleted through the admin
UI over time. There is **no unique index on `type`** — only the primary key — so nothing at the
database level prevents two rows sharing a type.

The tier rows are the whole configuration, and they all hang off 5/6/7/10:

| option id | offload | `total_price` | `price` | `enabled` |
|---|---|---|---|---|
| 18 | 5 (SELF) | 3000.00 | 0.00 | **0** |
| 12 | 6 (FF) | 1000.00 | **150.00** | 1 |
| 3 | 7 (ASSISTED) | 1000.00 | 6.00 | **0** |
| 16 | 10 (MECH) | 0.01 | 0.00 | **0** |
| 19 | 10 (MECH) | 3000.00 | 0.00 | 1 |

### 1.2 Why seven come back, not eight

`DeliveryController.js:422-425` fetches every row `.where("enabled", 1)` (no `is_deleted` filter —
that column is only ever filtered on `delivery_types`), then `getOffloads()`
(`app/Services/delivery/offload.js:46-71`) applies the tiers:

```js
let maxOptionPrice = 0;
let isEnabled = 1;
for (const option of globalOffload.offloadOptions || []) {
  if (totalPrice > option.total_price && option.total_price > maxOptionPrice) {
    globalOffload.price = option.price;
    maxOptionPrice = option.total_price;
    isEnabled = option.enabled;
  }
}
return isEnabled ? { id, type, name, details, price, sort_order } : null;
```

Only the **highest matching tier** wins (`option.total_price > maxOptionPrice`), so the result is
independent of row order. Row 10 carries tier 16 (`total_price 0.01`, `enabled 0`), so for any cart
above a penny and below £3,000 it returns `null` and is filtered out — **that is why QA sees 7 items
and not 8.** A row with *no* tiers can never be dropped and never be re-priced.

The final `sortAscending(…, "sort_order")` (`app/Utils/sort.js`) is a plain `Array.sort`, which is
stable in V8, so ties keep database order. That reproduces QA's payload exactly:

`1 (sort 1) → 4 (sort 1) → 5 (sort 1) → 3 (sort 2) → 7 (sort 2) → 2 (sort 3) → 6 (sort 3)`

### 1.3 The clients render it verbatim

Neither front end dedupes, and neither is at fault:

- **ff-uk-mobile** — `src/features/checkout/services/deliveryApi.ts:132-138` is a straight 1:1
  `.map`; `src/features/checkout/hooks/useDeliverySelection.ts:152-157` renders one row per entry.
  The list key is `option.id` (`components/OptionPickerSheet.tsx:39`), **not** `type`, so there is
  no React key collision — QA is seeing genuinely distinct rows.
- **gatsby-website** — `src/pages/checkout/delivery/index.js:135` takes the array as-is;
  `src/features/checkout/service-page/offload/offload-options.js:18-31` renders all of it, keyed by
  `option.id`.
- Nothing in any repo does `.find(o => o.type === …)` on offloads, so nothing silently collapses the
  pair. What *is* affected is the **default selection**, which is `list[0]` in both clients
  (`useDeliverySelection.ts:55`, `checkout/delivery/index.js:156`) — and `list[0]` is now id 1, the
  copy.

---

## 2. The stale prices are not the dead constant

QA reasonably joined two dots that are not connected. §6 of the delivery doc says the £30/£89 in
`OFFLOAD_PRICES` are stale and harmless because the constant is dead code. **That is still true** —
re-verified today:

```
$ grep -rn "OFFLOAD_PRICES" app/ start/
app/Utils/constants.js:69:exports.OFFLOAD_PRICES = {
```

One hit: the declaration. No reader anywhere in `app/`, `start/`, `database/`, `resources/` or
`test/`. (`OFFLOAD_TYPES` two lines above is equally unreferenced, and is missing
`CUSTOMER_OFFLOAD_MECH` entirely.)

The £30/£89 QA sees came out of the database. Both the constant and the duplicate rows are copies of
the **same pre-June-2026 price snapshot** — offloads 6 and 7 were repriced to £108/£43 on
**2026-06-02**, and every artefact created before that date carries the old pair. The matching
numbers are shared ancestry, not shared cause.

**§6 was also accurate when it was written.** It recorded four rows at ids 5/6/7/10 on 2026-08-20.
The copies did not exist yet — the earliest delivery referencing one is 2026-09-02 (§3.3).

---

## 3. Where rows 1–4 came from

### 3.1 It was not the admin UI, and it was not the price update logic

`admin-website-v2` has exactly one code path that inserts a `delivery_offloads` row — the explicit
**Create Offload** button
(`src/pages/content/delivery/delivery-offloads/delivery-offloads-page.jsx:41-53`), which posts
`{ type: 'UNTITLED', name: 'Untitled' }`. The edit screen
(`edit-delivery-offload.jsx:59-70` → `services/delivery/offload.js:40-43`) is always
`PUT /delivery/offload/:id` with the id from `useParams()`; there is no create fallback and no
clone button. Server side matches: `DeliveryOffloadController.update()` does `findBy('id', id)` then
`save()` — a pure update.

Two facts rule the admin UI out anyway:

- **`AUTO_INCREMENT` is 15.** A row created through the UI gets the next auto id — 11, 12, 13 … It
  is not possible to reach id 1–4 that way. Only an INSERT that names the ids can.
- **`created_at` on rows 1–4 is `2023-03-31 16:03:4x`.** Lucid's `create()` stamps the current
  time. These timestamps were supplied by the inserter.

### 3.2 The seed file is a byte-for-byte match

`database/seeds/DeliveryOffloadSeeder.js` hard-codes exactly these four rows:

```js
{ id: 2, type: "FIRST_FENCE_OFFLOAD", name: "First Fence Offload",
  details: "Our transport team will arrange suitable offloading facilities for you, …",
  sort_order: 3, price: 89.0, enabled: 1, is_deleted: 0,
  created_at: "2023-03-31 16:03:44", updated_at: "2023-03-31 16:03:46" },
{ id: 3, type: "ASSISTED_OFFLOAD",   …, sort_order: 2, price: 30.0, …,
  created_at: "2023-03-31 16:03:44", updated_at: "2023-03-31 16:03:47" },
```

Every column of dev rows 1–4 matches the file: ids, types, names, details, `sort_order`, prices,
`enabled`, `is_deleted`, and both timestamps down to the second. Nothing else in any of the five
repos contains those literals.

Before commit `30ee315` (2026-09-01) the seeder ended in a bare insert:

```js
await Database.table("delivery_offloads").insert(deliveryOffloads);
```

On a database whose offloads sit at 5/6/7/10, ids 1–4 are free, so that insert **succeeds silently**
and adds four rows. (On a database that already holds them at 1–4 it raises `ER_DUP_ENTRY` and
changes nothing — see §5.)

### 3.3 The timing window contains that run

`deliveries.offload` stores the chosen offload id, which dates the rows precisely:

| offload | rows | first used | last used |
|---|---|---|---|
| 5 (real SELF) | 40,668 | 2020-11-03 17:40 | **2026-09-01 11:30:06** |
| 6 (real FF) | 2,437 | 2020-11-03 13:17 | 2026-08-27 19:45 |
| 7 (real ASSISTED) | 1,666 | 2020-11-03 13:15 | 2026-08-27 19:41 |
| 10 (real MECH) | 1,705 | 2022-12-19 12:23 | 2026-08-25 18:44 |
| **1 (copy)** | **60** | **2026-09-02 06:24:08** | 2026-09-09 09:10 |
| **2 (copy)** | **2** | 2026-09-04 10:55 | 2026-09-04 12:58 |
| **3 (copy)** | **4** | 2026-09-07 18:14 | 2026-09-08 08:32 |

Real row 5 takes the last checkout at **2026-09-01 11:30**; copy row 1 takes the very next one at
**2026-09-02 06:24** and every one since. The rows were therefore inserted inside that window.
Commit `30ee315` — *"fix(seeds): make the seeders re-runnable without duplicating live data"* — is
dated **2026-09-01 18:25 +0300**, inside it.

### 3.4 The run is on record

`website-api/.generated_docs/epic-5-orders/NEXT-SESSION-PROMPT.md` logs that day's seeder work,
including a cleanup of duplicates written into `website_test`:

> **2026-09-01** · I WROTE 11 DUPLICATE ROWS INTO `website_test`. […] 5 duplicate
> `delivery_variables` types and 6 duplicate `depots`. […] Oleksandr ran the cleanup DELETEs;
> confirmed back to 26 and 7, zero duplicates.

Both of those tables are clean today — verified: `delivery_variables` 26 rows, `depots` 7 rows, no
duplicates. **`delivery_offloads` was never in that count**, because its four rows came from the
*earlier* bare-insert run rather than the "insert the missing ids" attempt that the snapshot was
taken around. They are the residue the cleanup did not know to look for.

---

## 4. Why only this one table

The same log entry explains it:

> **2026-09-01** · SEEDER ORDERING mattered […] `DatabaseSeeder` runs DeliveryOffload →
> DeliveryParam → Depot → NotificationChannel → NotificationType, so DeliveryParam's
> `ER_DUP_ENTRY` at position 2 BLOCKED the notification seeders at 4 and 5 from ever running.

`DeliveryOffloadSeeder` is **first** in `database/seeds/DatabaseSeeder.js:32`. Its four ids were free
on dev, so it committed. `DeliveryParamSeeder` runs second and inserts `delivery_variables` ids
1, 2, 3 … which already exist — `ER_DUP_ENTRY`, and the whole run aborts. `DepotSeeder` and
everything after it never executed.

That is exactly the state of the database today: `delivery_offloads` polluted, `delivery_variables`
and `depots` untouched by that run. It is not a coincidence — it is the seeder order.

---

## 5. Is production affected?

Three separate questions, and they have different answers.

### 5.1 Can a deploy do this to prod? No.

Neither pipeline touches the database:

- `.gitlab-ci.yml` (current, `only: master`): `git fetch` → `git reset --hard origin/master` →
  conditional `npm ci` → `pm2 restart 0`. No `adonis migration:run`, no `adonis seed`.
- `.gitlab-ci-old.yml`: `docker build` → `docker push` → `ecs deploy`. Same — nothing DB-facing.

`adonis seed --files=DatabaseSeeder.js` appears only in `README.md` and `CLAUDE.md` as a manual
setup step. Seeding prod requires a human running the command with prod credentials in `.env`.

### 5.2 Can it happen again? No.

Since `30ee315`, every one of these seeders goes through `seedIfEmpty`
(`app/Utils/seed-rows.js`), which counts the table and returns without writing if it holds any
rows. `delivery_offloads` is never empty in any deployed environment, so the seeder is now a no-op
there — deliberately, and with a spec covering it (`test/seed-rows.spec.js`).

### 5.3 Could prod already be in this state? Probably not — but unverified.

The mechanism needs ids 1–4 to be **free** in the target table. That is a property of dev, not a
universal one: the seed file itself is a snapshot of an environment where those rows sat at **ids
1, 2, 3, 4** with prices £89/£30 — authored 2026-01-21 by a First Fence developer, most plausibly
dumped from production. If prod does hold them at 1–4, a pre-fix seeder run there would have raised
`ER_DUP_ENTRY` on the first statement and written nothing.

Supporting evidence that the seed data is from a **different id-space than dev's**: the hardcoded
depot ids in `app/Utils/constants.js:80-84` (`GLASGOW_DEPOT_ID = 8`, `CANVEY_DEPOT_ID = 9`,
`BRISTOL_DEPOT_ID = 7`, `TIPTON_DEPOT_ID = 14`) match **none** of dev's depot ids (Glasgow 11,
Canvey 13, Bristol 14, Tipton 15). Dev's id-space has drifted from whatever the code was written
against.

I did **not** probe production to confirm. `POST /delivery` is read-only, but it sits behind the
`cartProducts` middleware (`start/routes/public.js:64-66`) and needs a real cart token, so checking
it that way would mean creating a cart on prod. The clean check is one read-only query:

```sql
SELECT id, type, name, price, enabled, is_deleted, created_at, updated_at
FROM delivery_offloads ORDER BY id;
```

Four rows → prod is fine. Eight rows, or any two rows sharing a `type`, → prod has the same problem
and §7 applies there too.

---

## 6. What actually breaks

### 6.1 Nothing validates the choice

Both clients send the offload back three ways — `offload` (id), `offloadType` (string) and
`offloadPrice` — and fold the price into `delivery.price` client-side
(`ff-uk-mobile/src/features/checkout/screens/CheckoutScreen.tsx:227-229`;
`gatsby-website/src/pages/checkout/contact-details/index.js:221-225`).

`OrderController.save()` (`app/Controllers/Http/OrderController.js:543-554`) writes all of it
straight from `request.post()`. There is no lookup against `delivery_offloads`, no re-pricing, no
check that the id and the type agree. The `StoreOrder` validator is commented out at
`start/routes/public.js:39`, and had no offload rules anyway. The charged amount comes from
`delivery.price` alone (`app/Utils/price-calculations.js:109-114`), i.e. from the browser.

**So picking a duplicate produces no error anywhere.** The order is created, charged at the client's
number, and the confirmation email looks perfect — the copies have byte-identical `name` and
`details`, so `resources/views/emails/order/delivery-info.edge:46-59` renders the right words next
to the wrong price. The admin order list is equally indistinguishable (`OrderController.js:190`
selects `delivery_offloads.name as offload`).

### 6.2 The business rules the copies bypass

For a cart **over £1,000** the intended offer is: Customer Offload free, First Fence Offload
**£150**, Assisted Offload **withdrawn**, Mechanical hidden. What `/delivery` returns today:

| id | shown as | should be |
|---|---|---|
| 1 | Customer Offload — free | (duplicate of 5) |
| 4 | Customer Offload (Mechanical) — free | **hidden above £0.01** |
| 5 | Customer Offload — free | ✅ |
| 3 | **Assisted Offload — £30** | **not offered at all** |
| 2 | **First Fence Offload — £89** | **£150** |
| 6 | First Fence Offload — £150 | ✅ |

Over **£3,000** it also keeps `SELF_OFFLOAD` alive through id 1, which tier 18 is configured to
withdraw.

There is one more escape hatch. The per-product override in `getOffloads()`
(`offload.js:33-45`) short-circuits before the tier loop and is keyed by `offload_id`, so it only
ever hits one row of a pair. Dev has a single such override — product `68a44be02872df63886a5ae3`,
offload 6, **£500**. For a cart containing that product, id 6 is forced to £500 while **id 2 sails
through at £89**.

### 6.3 Measured damage on dev

66 deliveries created since 2026-09-02 reference the copies, and the pattern is still running today:

| offload | stored `offloadPrice` | rows | correct price | shortfall |
|---|---|---|---|---|
| 1 | `NULL` / 0.00 | 60 | free | — (but bypasses the £3,000 withdrawal) |
| 2 | £89.00 | 2 | £108.00 | −£38 |
| 3 | £30.00 | 2 | £43.00 | −£26 |
| 3 | £60.00 (2 vehicles) | 2 | £86.00 | −£52 |

**£116 under-charged across 6 dev orders in a week.** On production traffic the same defect would
scale with volume.

Downstream, `app/Utils/salesforce-quotes.js:278-322` picks the CPQ SKU from `offload_type`, so a
copy resolves to the **correct SKU at the wrong price** — id 2 and id 6 both map to `SERV-LOA-0001`,
with `Target_Price__c` set to whatever the client sent. Nothing on the Salesforce side can spot it;
it looks like a legitimately discounted line. The delivery line absorbs the difference
(`salesforce-quotes.js:233-234`: `deliveryPriceWithoutOffload = price - offloadPrice`), so the quote
total still ties out while the split between SKUs is wrong.

Two `/delivery` post-processors also operate on the array by type and so hit both copies —
`app/Services/delivery/residential-gate-delivery.js:61-63` keeps every `FIRST_FENCE_OFFLOAD` and
zeroes its price, which for a residential gate order returns **two identical £0 "First Fence
Offload" rows** to the client.

---

## 7. Fix

**Immediate, dev — one statement, no FK risk:**

```sql
UPDATE delivery_offloads SET enabled = 0, is_deleted = 1 WHERE id IN (1, 2, 3, 4);
```

`/delivery` filters on `enabled = 1` (`DeliveryController.js:424`), so the copies disappear from
every client at once, and the 66 existing `deliveries` rows keep their foreign key intact.

**Do not `DELETE` them.** `deliveries_offload_foreign` is `ON DELETE SET NULL` — a delete would
succeed silently and null `deliveries.offload` on 66 rows, leaving `offload_type` and `offloadPrice`
orphaned behind it. If those rows matter, repoint them first
(`UPDATE deliveries SET offload = 5 WHERE offload = 1`, and 2→6, 3→7), then disable.

**Then check production** with the query in §5.3 before assuming it is clean.

**Worth doing regardless of this incident:**

| # | Change | Why |
|---|---|---|
| 1 | Unique constraint or a `store()` guard on `delivery_offloads.type` | Nothing at any layer prevents two rows sharing a type — not the DB, not the model, not the controller |
| 2 | Server-side re-price on `POST /order/create`: look the offload up by id, use *its* price | The offload price is currently whatever the browser says; this is the actual revenue exposure, duplicates or not |
| 3 | Add a **Type** column to the admin offload list (`delivery-offloads-page.jsx:90-115`) | It shows ID, Name, Price, Status, Sort Order — two rows with the same type are invisible there today |
| 4 | Delete `OFFLOAD_PRICES` and `OFFLOAD_TYPES` from `app/Utils/constants.js:63-73` | Dead, stale, and — as this ticket demonstrates — actively misleading when someone is debugging a £30/£89 |
| 5 | `.toUpperCase()` in `salesforce-quotes.js:282-308` | Unrelated pre-existing bug: 1,539 dev deliveries carry a lowercase `offload_type`, and the case-sensitive `===` means **no offload line at all** reaches the CPQ quote. The delivery-SKU path at `:213-214` already normalises; the offload branch does not |

---

## Appendix — queries used

```bash
# all read-only, via the local mysql client container against the dev remote
docker exec -e MYSQL_PWD='<pass>' firstfence-mysql \
  mysql --connect-timeout=20 -h 54.171.181.199 -u mobile_dev website_test -e "…"
```

```sql
-- the 8 rows
SELECT id, CAST(type AS BINARY) AS type, name, sort_order, price,
       enabled, is_deleted, created_at, updated_at
FROM delivery_offloads ORDER BY id;

-- the tier configuration that only the real rows have
SELECT * FROM delivery_offload_options ORDER BY id;

-- dates the copies: real ids stop, copies start
SELECT offload, COUNT(*) AS rows_, MIN(created_at) AS first_used, MAX(created_at) AS last_used
FROM deliveries GROUP BY offload ORDER BY offload;

-- the under-charges
SELECT offload, offloadPrice, COUNT(*) FROM deliveries
WHERE created_at > '2026-08-01' GROUP BY offload, offloadPrice ORDER BY offload, offloadPrice;

-- proves the admin UI could not have made ids 1-4
SELECT TABLE_NAME, AUTO_INCREMENT FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'website_test' AND TABLE_NAME = 'delivery_offloads';

-- FK delete rules, before anyone reaches for DELETE
SELECT CONSTRAINT_NAME, DELETE_RULE, TABLE_NAME FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'website_test' AND REFERENCED_TABLE_NAME = 'delivery_offloads';
```

Companion docs: [`delivery-types-and-options.md`](./delivery-types-and-options.md) (§6 covers the
offload field and the case-sensitivity bugs in `offload_type`).
