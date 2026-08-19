# Trade (Credit) Account Approval Status — Is It Knowable From the App?

Answer to the RN team's question: *"is there a way to know the status of a trading (credit)
account application — approved / pending / rejected — so we can show a 'Pending' state in the UI?"*

---

## TL;DR

**No. There is no application status anywhere in the stack** — not in `website-api` (MySQL), not
in SAP, and not readably in Salesforce. Approval is a manual decision whose only recorded outcome
is a credit line appearing in SAP.

- **The only signal is binary.** `hasCreditAccount` on
  `GET /api/customer-portal/customers/credit-account` is just `OCRD.CreditLine > 0`.
  `false` means *never applied* **or** *pending* **or** *rejected* **or** *cash customer* — the
  four are indistinguishable. (A user with no `CardCode` gets `404`, not `false`.) → §1, §2
- **Applications aren't stored queryably.** The website's real application flow is an external
  **`forms.office.com`** link that never touches `website-api`. The two tables that could hold
  something (`trade_signatures`, `contact_forms`) have **no status column, no `user_id`**, are
  write-only, and have no admin screen. There is no `credit_applications` table. → §3
- **Don't build Salesforce polling.** Even if the read worked, most applicants never create a
  Salesforce record (Office Forms), and Task closure ≠ approved. → §5.3
- **Two app bugs to fix regardless of what's decided:**
  1. `RequestCreditAccountScreen.tsx:15` sends `recaptchaToken: 'mobile-app'` → the endpoint
     `400`s, so **the application is never submitted and the on-device pending flag is never
     set**. → §4.1
  2. `normalizeUser.ts:22` derives `creditStatus` from `CardCode`, which is auto-backfilled for
     anyone who has ever ordered → **cash customers get trade UI**. Branch on `hasCreditAccount`
     instead. → §2.1
- **The current on-device "Pending review" flag matches what was specified** (FIR-24 AC2,
  FIR-121: pending-only, no live status). Fine for V1, with three caveats: per-device, invisible
  to the web portal, and **it can never clear on rejection**. → §4
- **For a real status: Option B** — a `credit_applications` table in `website-api`, surfaced as an
  `application` block on the existing endpoint, plus an admin screen for sales to resolve it.
  New work; only covers app-originated applications until the website form moves off Office
  Forms. The crux is **who resolves it and where** — sales won't work in a screen they don't
  already use. → §5.2, §7

---

Companion docs:
[`credit-account-flow.md`](./credit-account-flow.md) (the read-only balance endpoint) and
[`../orders-and-payments/order-and-payment-status-flow-trade-credit.md`](../orders-and-payments/order-and-payment-status-flow-trade-credit.md)
(what happens once a customer *has* credit and pays on account).

---

## 1. What "approved" means in the system today

There is exactly one signal, and it lives in SAP, not in First Fence's own database:

> **`OCRD.CreditLine > 0`** — the business partner has a credit line ⇒ the trade/credit account
> is approved and live.

That single field is the de facto gate, and it is read in two independent places — which is why
it is safe to treat it as *the* definition rather than one endpoint's opinion:

| Where | Code | Use |
|-------|------|-----|
| Credit summary endpoint | `app/Controllers/Http/Portal/CustomerCreditAccountController.js:29` | `if (!creditLimit) return { hasCreditAccount: false }` |
| Pay-on-account at checkout | `app/Controllers/Http/CreditPaymentController.js:130` | same check gates whether credit payment is allowed at all |

Both reach SAP through `SAPService.getProfile(cardCode)`
(`app/Services/SAPService.js:4`), which is a plain
`SELECT * FROM "OCRD" WHERE "CardCode" = ? LIMIT 1`.

**Useful detail:** because it is `SELECT *`, *every* `OCRD` column is already in memory on the
API side. Exposing additional business-partner fields needs **no new SAP query and no SAP-side
work** — only a change to the response mapping. That matters for option D below.

---

## 2. Why `hasCreditAccount: false` cannot be used as "pending"

`GET /api/customer-portal/customers/credit-account` collapses four genuinely different
situations into the same two responses:

| Real-world situation | What the endpoint returns |
|----------------------|---------------------------|
| Never applied | `200 { hasCreditAccount: false }` |
| Applied, under review | `200 { hasCreditAccount: false }` |
| Applied, **rejected** | `200 { hasCreditAccount: false }` |
| Cash customer, no interest in credit | `200 { hasCreditAccount: false }` |
| Approved and live | `200 { hasCreditAccount: true, ... }` |
| No `CardCode` on the user record | `404 { message: "Card code not found" }` |

So the response is a perfectly good **"is trade now?"** flag and carries **zero** information
about an application in flight. Note also the 404: a brand-new user who has never ordered gets a
**404, not `hasCreditAccount: false`** — the app must treat that as "no credit account", not as an
error state.

### 2.1 The `CardCode` heuristic in the app is also not a credit signal

`ff-uk-mobile/src/features/auth/utils/normalizeUser.ts:22`:

```ts
const creditStatus: CreditStatus = raw.CardCode ? 'credit' : 'none';
```

This is currently wrong in a way that will show trade UI to ordinary cash customers.
`CardCode` means *"linked to an SAP business partner"*, not *"has credit"*, and it is
**auto-backfilled for anyone who has ever placed an order that reached SAP**:

`app/Controllers/Http/UserController.js:76-116` — on every `getAuthenticatedUser` (`/me`) call,
if the user has no `CardCode`, the API finds their latest order with a `SAP_ORDER_ID`, resolves
the card code via `SAPService.getCardCodeByDocNum`, and saves it. Admins can also set it by hand
(`UserController.js:154`, `Portal/CustomerController.js:387`, and the admin panel's users /
customers pages).

**Fix regardless of which option below is chosen:** derive trade-ness from
`hasCreditAccount` on the credit-account endpoint, never from `CardCode` presence.
`CardCode` gates *"can we even ask SAP about this user"*, nothing more.

---

## 3. What actually happens when someone applies today

There are **three** application paths in the codebase, and they leave three different traces.
None of them records a status.

### 3.1 The live website path — Microsoft Office Forms (no backend trace at all)

`gatsby-website/src/features/trade-account/open-account-box/open-account-box.js:23` — the
"Request Application Form" button on `/trade-account` is an **external link to
`forms.office.com`**. The secondary CTA ("Get in Touch", and the portal's "Apply" button in
`src/features/portal/trade-account-box/trade-account-box.jsx:73`) just routes to `/contact-us`.

This is the important correction to the premise in the original question: **the website's trade
account application does not go to Salesforce, and does not touch `website-api` at all.** It goes
into a Microsoft Forms response sheet that someone on the sales side works through manually.
Nothing in any First Fence system knows an application was submitted.

### 3.2 `POST /tradesignature` — the legacy path (orphaned)

`start/routes/public.js:77` → `TradeSignatureController.store`. Writes `email` +
`email_optional` to MySQL `trade_signatures` and fires `send::tradesignature`, which emails
**account.applications@firstfence.co.uk** with the subject *"New Account Application Form
Requested"* (`app/Listeners/TradeSignature.js:9-24`).

- Schema (`database/migrations/1612432009728_trade_schema.js`): `id`, `email`, `email_optional`,
  `timestamps`. **No status column, no user_id, no company.**
- **No front-end calls it any more** — `tradesignature` appears nowhere in `gatsby-website`,
  `admin-website-v2`, or `ff-uk-mobile`. It is dead but still routable.
- No admin UI reads `trade_signatures`.

### 3.3 `POST /sendContactForm` — the generic contact form (what mobile currently reuses)

`start/routes/public.js:79` → `ContactFormController.store`, behind the `verifyRecaptcha`
middleware. It writes a `contact_forms` row and fires two events: an email, and
`salesforce::createTask`.

`app/Listeners/Salesforce.js:11-42` builds a Salesforce **Task** with
`Status: "Not Started"` and POSTs it to a custom Apex endpoint `${SF_API_URL}/taskFromContactForm`.

This is the only path that creates a Salesforce record with a **status field on it** — which is
why it is the basis of option C below. Two caveats before relying on it:

- `contact_forms` itself has **no status column** and no admin screen
  (`database/migrations/1616150529690_contact_form_schema.js`); it is write-only.
- The write is **fire-and-forget and silently swallowed**: any failure is caught and logged to
  Grafana (`Salesforce.js:38-41`), so the HTTP caller always sees success even when nothing
  reached Salesforce.

### 3.4 Summary

| Path | Reaches Salesforce? | Server-side record | Status field |
|------|--------------------|--------------------|--------------|
| Office Forms link (**the live website flow**) | ❌ | ❌ none | ❌ |
| `POST /tradesignature` (orphaned) | ❌ (email only) | MySQL `trade_signatures` | ❌ |
| `POST /sendContactForm` | ✅ Task (but see §5.3) | MySQL `contact_forms` | ⚠️ on the SF Task only |

---

## 4. What the mobile app does today

The app has already shipped a **device-local** stand-in for the missing status, and it is honest
about why — `ff-uk-mobile/src/features/credit/utils/creditRequestStorage.ts`:

```ts
// The contact-form endpoint doesn't expose the application's review status, so
// "pending review" is tracked on-device per user until sales resolves it in
// Salesforce (at which point the credit-account endpoint starts returning data).
```

`RequestCreditAccountScreen.tsx` submits to `/sendContactForm` with
`nature: 'Trade credit account application'`, then calls `markCreditRequestPending(user.id)`,
which writes a timestamp into `expo-secure-store`. `CreditBalanceCard.tsx` renders the
`pending` variant ("Pending review" `AlertBox`, amounts blanked to `£ –`) off that flag.

Limits of the on-device approach, worth stating to the client explicitly:

- **Per-device.** Reinstall, new phone, or logging in on a second device loses the pending state.
- **Never clears on rejection.** It only stops mattering once `hasCreditAccount` flips to `true`.
  A rejected applicant sees "Pending review" forever.
- **Invisible to web.** The Gatsby portal shows the marketing "Apply" box in the same situation,
  so the two front ends disagree.

### 4.1 ⚠️ The current submission almost certainly fails

`RequestCreditAccountScreen.tsx:14-15`:

```ts
// TODO: the contact-form endpoint sits behind Google reCAPTCHA
const RECAPTCHA_PLACEHOLDER = 'mobile-app';
```

`/sendContactForm` is wrapped in `App/Middleware/CaptchaVerify`, which posts the token to
Google's `siteverify` and returns **`400 { message: "Captcha verification failed" }`** unless
`success` is true. The literal string `'mobile-app'` will not verify. There is no reCAPTCHA
client in the app — the only match in `ff-uk-mobile` is the `RecaptchaInterop` iOS pod in
`app.config.ts:159`, which is a transitive Google/Firebase build shim, not a token provider.

Because `markCreditRequestPending` runs only after `.unwrap()` resolves, the consequence is that
**the application is never submitted and the pending flag is never even set** — the user sees the
error alert. This needs fixing (real reCAPTCHA token, or a mobile-exempt endpoint) *before* any
of the status work below is worth doing.

---

## 5. Options for a real status, ranked

### 5.1 Option A — keep it on-device (status quo)

Zero backend work; matches what Linear already asked for. **FIR-24 AC2 is literally
"No live status: the app shows only 'Pending' — no live status updates from the sales team."**
Accept the three limits in §4 and fix the reCAPTCHA bug.

Good enough if the client confirms they are happy with a cosmetic, per-device pending badge.

### 5.2 Option B — application record in `website-api` (recommended)

A small owned table, e.g. `credit_applications`: `user_id`, `company_name`, submitted fields,
`status` enum (`pending` / `approved` / `rejected` / `withdrawn`), `submitted_at`,
`resolved_at`, `notes`. Then:

- `POST /api/customer-portal/customers/credit-account/applications` — submit (replaces the
  contact-form reuse, and can be exempted from reCAPTCHA since it is behind `auth` + `customer`).
- Extend the existing credit-account response with an `application` block, so one call answers
  everything:

  ```json
  { "hasCreditAccount": false,
    "application": { "status": "pending", "submittedAt": "2026-08-01T10:12:00.000Z" } }
  ```

- Keep the `salesforce::createTask` fire-and-forget write so sales still gets their queue item.
- Add a screen in `admin-website-v2` where sales flips the status. **This is the crux:** a real
  status needs a human to set it somewhere First Fence staff actually work. Without that the
  column just stays `pending` forever and Option B degrades into Option A with extra steps.

Why this is the recommendation: it is entirely within our control, it is cross-device and
cross-client (the Gatsby portal gets the same state for free), it survives reinstalls, and it can
express **rejected** — which no other option can today. It also does not depend on First Fence's
Salesforce team shipping anything.

Backwards compatible: existing clients ignore the new `application` key.

### 5.3 Option C — read the Salesforce Task status back

`SalesforceService` already has a generic `query(soql)` (`app/Services/SalesforceService.js:31`)
using `@jsforce/jsforce-node`, so reading is technically available today. A Task created by
`createTask` carries `Status` ("Not Started" → "In Progress" → "Completed") and
`From_Email__c`, so in principle:

```sql
SELECT Id, Status, Subject, CreatedDate FROM Task
WHERE From_Email__c = '<user email>' AND Subject = 'Trade credit account application'
ORDER BY CreatedDate DESC LIMIT 1
```

Problems, roughly in order of severity:

1. **It misses most applicants.** The real website flow is Office Forms (§3.1), which creates no
   Task. Only mobile-originated applications would ever be found.
2. **"Completed" ≠ "approved".** Task closure means a rep dealt with the enquiry — approved,
   rejected, or duplicate. There is no approval outcome to read, so the app still cannot say
   "rejected".
3. **Matching is fragile** — free-text `Subject` (it comes straight from the form's `nature`
   field) plus email equality, with no application ID anywhere.
4. **Config is unverified.** `SF_API_URL` and `SALESFORCE_ACCESS_TOKEN` — the two variables
   `createTask` depends on — are **absent from `.env.example`**, and `createTask` is the only
   Salesforce caller in the codebase using a static `process.env.SALESFORCE_ACCESS_TOKEN` instead
   of `SalesforceTokenService.getToken()`. Every other caller (`CreditPaymentController.js:270`,
   `Portal/CustomerQuoteController.js:110`, `Portal/TicketController.js`) refreshes OAuth
   properly. So whether tasks are being created at all in the deployed environments must be
   confirmed before building on this.
5. **Sandbox by default** — `SF_BASEURL` in `.env.example` is
   `https://firstfence--uat.sandbox.my.salesforce.com`, and `SalesforceService` falls back to
   `https://test.salesforce.com`. Production endpoints and API call limits under mobile load
   need confirming.
6. Per-request SOQL on a hot path (the credit panel) adds latency and burns API quota.

Viable only as an **enhancement on top of Option B**, and only if Salesforce adds a real
approval-outcome field and the website form is migrated off Office Forms.

### 5.4 Option D — an SAP-side signal

Since `getProfile` is already `SELECT *`, any `OCRD` column can be surfaced with a response-mapping
change alone. In stock SAP Business One, `OCRD.CardType` distinguishes `C`ustomer / `S`upplier /
`L`ead, and there are `validFor` / `frozenFor` flags plus user-defined `U_*` columns — so a
convention like *"lead created when the application form arrives, promoted to customer on
approval"* could in principle yield a genuine pending signal.

**Unverified.** We cannot confirm whether First Fence populate `CardType`, or any `U_*` field,
that way — nothing in this codebase reads them (the only `U_*` column used anywhere is
`U_FLN_MBOX` in `SAPService.js:50`, an orders filter). This needs a question to First Fence /
their SAP partner, not a code change. Treat as a **research task**, not an option to plan around.

### 5.5 Comparison

| | A · on-device | B · own table | C · Salesforce read-back | D · SAP field |
|---|---|---|---|---|
| Backend work | none | small–medium (+ admin UI) | medium | unknown |
| Cross-device / cross-client | ❌ | ✅ | ✅ | ✅ |
| Can express **rejected** | ❌ | ✅ | ❌ | maybe |
| Covers website applicants | ❌ | ❌ (until web migrates) | ❌ | ✅ |
| Depends on third parties | no | no | Salesforce team | SAP partner |
| Blocked on unknowns | no | no | §5.3 config + SF schema | everything |

---

## 6. Recommended reply to the RN team

1. **There is no approval-status API and nothing to poll.** The only server-side truth is
   `hasCreditAccount` on `GET /api/customer-portal/customers/credit-account`, which is
   `OCRD.CreditLine > 0` — a binary "is trade now", with no in-flight or rejected state.
2. **This is by design in the current scope, not an oversight.** FIR-24 AC2 and FIR-121 both
   specify pending-only with no live status; approval stays a manual sales decision and
   "the user becomes trade on next data refresh". Keep the current on-device flag for now.
3. **Two things to fix in the app regardless:**
   - the reCAPTCHA placeholder (§4.1) — the application submission is currently failing;
   - `creditStatus` derived from `CardCode` (§2.1) — it will show trade UI to cash customers.
     Switch to `hasCreditAccount`, and treat the `404 Card code not found` as "no credit account".
4. **Don't build polling against Salesforce.** Even if the read worked, the website's real
   application flow is a Microsoft Forms link that creates no Salesforce record, so most
   applicants would never appear.
5. **If the client wants a trustworthy pending/approved/rejected state, that is Option B** — a
   `credit_applications` table in `website-api`, surfaced as an `application` block on the
   existing credit-account response, plus an admin screen for sales to resolve it. Estimate it as
   new work (backend + admin UI), and note it only covers app-originated applications until the
   website form moves off Office Forms.

## 7. Decisions needed from First Fence

Meeting 1 / Q2 raised three of these and **none has been answered** — Meeting 3 did not return to
them, and FIR-24 / FIR-120 / FIR-121 are all still `Backlog`.

1. Is a **cosmetic per-device "Pending review"** acceptable for V1 (Option A), or do they want a
   real tracked status (Option B)?
2. If real: **who resolves it, and where?** Sales will not work in a screen they do not already
   use — this decides whether Option B works or rots.
3. Should the **website's Office Forms application be replaced** by the same endpoint, so web and
   app applicants are visible in one place?
4. Does Salesforce hold, or can it hold, an **approval outcome** (approved / rejected) — not just
   Task closure — and can it push or expose it?
5. Does SAP mark applicants before approval (`OCRD.CardType = 'L'`, a `U_*` flag, anything)? (§5.4)
6. Confirm `SF_API_URL` / `SALESFORCE_ACCESS_TOKEN` are actually set in staging and production,
   and that contact-form tasks are landing in the sales queue today. (§5.3 item 4)

---

## 8. Code reference index

**website-api**
- `app/Controllers/Http/Portal/CustomerCreditAccountController.js` — the credit summary endpoint
- `app/Controllers/Http/CreditPaymentController.js:122-184` — the second `CreditLine` gate
- `app/Services/SAPService.js:4` — `getProfile`, `SELECT * FROM OCRD`
- `app/Controllers/Http/UserController.js:76-116` — `CardCode` auto-backfill from SAP orders
- `app/Controllers/Http/ContactFormController.js` — `/sendContactForm`
- `app/Controllers/Http/TradeSignatureController.js` + `app/Listeners/TradeSignature.js` — the
  orphaned `/tradesignature` application-request path
- `app/Listeners/Salesforce.js:11-42` — `createTask` (Salesforce Task, `Status: "Not Started"`)
- `app/Services/SalesforceService.js:31` — generic SOQL `query()` (read capability already exists)
- `app/Services/SalesforceTokenService.js` — OAuth token service that `createTask` does *not* use
- `app/Middleware/CaptchaVerify.js` — the reCAPTCHA gate on `/sendContactForm`
- `start/routes/customer-portal.js:7-10`, `start/routes/public.js:77-81` — routes
- `database/migrations/1612432009728_trade_schema.js`,
  `database/migrations/1616150529690_contact_form_schema.js` — both status-less

**ff-uk-mobile**
- `src/features/credit/utils/creditRequestStorage.ts` — the on-device pending flag
- `src/features/credit/screens/RequestCreditAccountScreen.tsx` — application form (reCAPTCHA bug)
- `src/features/credit/components/CreditBalanceCard.tsx` — the `pending` UI variant
- `src/features/credit/services/creditApi.ts`, `src/features/wallet/services/walletApi.ts` — API layer
- `src/features/auth/utils/normalizeUser.ts:22` — the incorrect `CardCode` → `creditStatus` derivation

**gatsby-website**
- `src/features/trade-account/open-account-box/open-account-box.js:23` — the Office Forms link
- `src/features/portal/trade-account-box/trade-account-box.jsx` — `hasCreditAccount` branch, no pending state

**Not involved:** `cdn-graphql-v2` has no credit/SAP/Salesforce concepts at all.
`admin-website-v2` can edit a user's `CardCode` (`src/pages/users/users-page.jsx:278`,
`src/pages/customers/customers-page.jsx:449`) but has no credit-application screen.

**Linear:** [FIR-24](https://linear.app/first-fence/issue/FIR-24) (apply, 5pts, Backlog) ·
[FIR-120](https://linear.app/first-fence/issue/FIR-120) (form → Salesforce, Backlog) ·
[FIR-121](https://linear.app/first-fence/issue/FIR-121) ("Pending review" panel, Backlog) ·
[FIR-25](https://linear.app/first-fence/issue/FIR-25) (limit increase, Backlog) ·
[FIR-96](https://linear.app/first-fence/issue/FIR-96) (company details + apply CTA, QA) ·
[FIR-54](https://linear.app/first-fence/issue/FIR-54) (SAP/Salesforce overview, Completed) ·
Meeting 1 / Q2 in `website-api/.generated_docs/FirstFence-Meetings.md:28-46`.
