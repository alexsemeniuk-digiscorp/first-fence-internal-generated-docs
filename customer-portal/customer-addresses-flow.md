# Customer Profile Addresses — FE Spec

CRUD for the logged-in customer's **address book**. Addresses live in the
`addresses` table and are linked to the customer's profile via a pivot
(`customer_profile_addresses`) that also carries the **label** and the
**default billing / shipping** flags.

Base path: `/api/customer-portal`

- **Middleware (all routes):** `auth` + `customer`
- The customer/profile is always resolved from the auth token — you never pass a
  profile id. The profile is auto-created if missing (`ensureForUser`).

| Method | Path | Action |
|--------|------|--------|
| `POST`   | `/customer-profiles/addresses` | List (paginated) |
| `POST`   | `/customer-profiles/addresses/save` | Create **or** update (upsert) |
| `DELETE` | `/customer-profiles/addresses/:addressId` | Remove from profile |

> ⚠️ **The list endpoint is `POST`, not `GET`** (pagination/search params go in the
> body). Don't let the `GET` in the controller's doc-comments mislead you.

---

## 1. List addresses — `POST /customer-profiles/addresses`

**Body (all optional):**

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `currentPage` | number | 1 | page number |
| `pageLimit` | number | 10 | items per page |
| `search` | string | — | matches org, all street lines, city, county, post_code, label |

**Success — 200** (Lucid paginator shape):
```json
{
  "total": 3,
  "perPage": 10,
  "page": 1,
  "lastPage": 1,
  "data": [
    {
      "pivot_id": 12,
      "customer_profile_id": 5,
      "address_id": 34,
      "label": "Main Address",
      "is_default_billing": 1,
      "is_default_shipping": 1,
      "pivot_created_at": "2026-01-10 09:00:00",
      "pivot_updated_at": "2026-01-10 09:00:00",
      "organisation": "Acme Ltd",
      "street_address_line1": "1 High St",
      "street_address_line2": null,
      "street_address_line3": null,
      "street_address_line4": null,
      "city": "Leeds",
      "post_code": "LS1 1AA",
      "county": "West Yorkshire",
      "country": "United Kingdom"
    }
  ]
}
```
- `pivot_id` = the `customer_profile_addresses` row id; `address_id` = the
  `addresses` row id. **Use `address_id` for save (update) and delete**, not
  `pivot_id`.
- `is_default_billing` / `is_default_shipping` come back as `1`/`0` (MySQL boolean).
- Ordered by pivot `created_at` desc (newest first).

**Errors:** `404 { "error": "Customer profile not found" }`,
`500 { "error": "Unable to fetch addresses" }`.

---

## 2. Create / update — `POST /customer-profiles/addresses/save`

Upsert. **Omit `address_id` to create; include it to update** that address.

**Body:**

| Field | Type | Notes |
|-------|------|-------|
| `address_id` | number | omit → create new; present → update existing address |
| `label` | string | pivot label (e.g. "Warehouse"); falsy → stored as `null` |
| `organisation` | string | |
| `street_address_line1`–`4` | string | line1 typically the main line |
| `city` | string | |
| `county` | string | |
| `post_code` | string | |
| `country` | string | |
| `is_default_billing` | boolean | if truthy, clears this flag on all other addresses first |
| `is_default_shipping` | boolean | if truthy, clears this flag on all other addresses first |

Behaviour:
- Runs in a transaction: writes the `addresses` row, then upserts the pivot row.
- **Defaults are exclusive** — setting `is_default_billing`/`is_default_shipping`
  truthy unsets it on every other address for this profile, so exactly one stays
  default. If you send them falsy/omitted, this address is set to **not** default
  (the flags default to `false` on each save).
- No field validation — send whatever the form has; empty fields persist as null.

**Success — 200:** returns the customer's **full address list** (the profile's
related addresses), so the FE can refresh straight from the response. (This is the
model relation, not the paginated/joined shape from endpoint #1.)

**Errors:** `404 { "error": "Customer profile not found" }`,
`400 { "error": "Unable to save address" }` (transaction failed),
`500 { "error": "Unable to save address" }`.

---

## 3. Remove — `DELETE /customer-profiles/addresses/:addressId`

`:addressId` is the **`address_id`** (not the pivot id).

Behaviour:
- Detaches the address from this profile (deletes the pivot row).
- If no other profile references that address, the `addresses` record itself is
  also deleted. Otherwise the shared address is kept.

**Success — 204** No Content (empty body).

**Errors:**
- `404 { "error": "Address not linked to this profile" }` — no pivot for this
  profile + address.
- `404 { "error": "Customer profile not found" }`.
- `500 { "error": "Unable to remove address" }`.

---

## FE notes

- **`address_id` is the handle** for update and delete — read it from the list's
  `address_id` field (not `pivot_id`).
- Booleans come back as `1`/`0` from the list endpoint — coerce when rendering toggles.
- **Save returns a different shape than list** (model relation vs. paginated join).
  If you rely on `pivot_id` / joined columns, re-fetch via endpoint #1 after saving
  rather than trusting the save response shape.
- Only one address can be default-billing and one default-shipping at a time — the
  backend enforces exclusivity, so the UI can treat it as a radio, not a checkbox.
- All error bodies use the flat `{ "error": "..." }` shape (consistent here).