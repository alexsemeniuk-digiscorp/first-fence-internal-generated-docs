# Customer Credit Account — FE Spec

Returns the logged-in customer's **credit account** summary (credit limit and
available balance), sourced live from SAP HANA.

Endpoint: `GET /api/customer-portal/customers/credit-account`

- **Middleware:** `auth` + `customer`
- **Request body / params:** none — the customer is taken from the auth token.

---

## How it works

1. Resolves the authenticated user and reads their `CardCode` (SAP customer code).
2. Fetches the SAP business-partner profile (`OCRD`) for that `CardCode`.
3. If the profile has no credit line, returns `hasCreditAccount: false`.
4. Otherwise computes available balance and returns the summary:

```
availableBalance = creditLimit − (balance + deliveryNotesBalance + ordersBalance)
```
(where `balance = OCRD.Balance`, `deliveryNotesBalance = OCRD.DNotesBal`,
`ordersBalance = OCRD.OrdersBal`, `creditLimit = OCRD.CreditLine`.)

---

## Responses

### 200 — has a credit account
```json
{
  "hasCreditAccount": true,
  "companyName": "Acme Fencing Ltd",
  "creditLimit": 5000,
  "availableBalance": 3200.5
}
```
| Field | Type | Notes |
|-------|------|-------|
| `hasCreditAccount` | boolean | `true` here |
| `companyName` | string | from SAP `CardName`, cleaned (asterisks stripped, capitalised) |
| `creditLimit` | number | SAP `CreditLine` |
| `availableBalance` | number | `creditLimit − (balance + deliveryNotes + orders)`; can be **negative** if over limit |

### 200 — no credit account
```json
{ "hasCreditAccount": false }
```
Returned when the customer exists in SAP but has no credit line (`CreditLine` is
0/empty). **FE must handle this shape** — only `hasCreditAccount` is present; the
other fields are absent.

### 404 — no card code
```json
{ "message": "Card code not found" }
```
The user account has no `CardCode` (not yet linked to an SAP customer).

### 404 — profile not found
```json
{ "message": "Profile not found" }
```
Documented for completeness. In practice this rarely fires — the SAP lookup returns
an empty object rather than null when nothing matches, which falls through to the
`hasCreditAccount: false` branch instead.

### 500 — server error
```json
{ "error": { "message": "Internal server error" } }
```
Any failure (e.g. SAP HANA unavailable). Note the nested `error.message` shape here,
unlike the flat `message` used by the 404s.

---

## FE notes

- **Always branch on `hasCreditAccount` first.** When `false`, do not read
  `creditLimit` / `availableBalance` — they aren't returned.
- `availableBalance` can be **negative** (customer over their limit) — don't assume ≥ 0.
- Amounts are plain numbers (no currency/formatting) — format client-side (GBP).
- Data is **live from SAP HANA** on every call, so expect occasional latency and the
  possibility of a `500` if SAP is down — show a graceful retry/fallback.
- Error response shapes are inconsistent: 404s use `{ "message": ... }`, the 500 uses
  `{ "error": { "message": ... } }` — handle both.