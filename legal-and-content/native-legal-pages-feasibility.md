# Native Legal Pages in the Mobile App — Where the Content Comes From, and Whether to Build Them

Answer to the RN team's question: *"we want to build native legal pages — where does all this
data come from, and is it possible (and worth it)?"*

Scope covered: `https://firstfence.co.uk/legal-info`, `/privacy-policy`, `/refunds`, and the three
T&C PDFs on `media.firstfence.co.uk/docs/`.

---

## TL;DR

**Yes, it's possible — and cheaper than expected for the web pages, impossible for the PDFs
without new content work.** The split between those two halves is the whole story.

- **The web legal pages are already a CMS-backed API the app can call today.** All five live in
  one Mongo collection (`cdn.cmspages`) and are served by `cdn-graphql-v2` via `allCMSPages`,
  **unauthenticated**. The mobile app already talks to that exact API with no auth headers. Zero
  backend work needed. → §1.1, §4
- **The app already has the renderer.** `react-native-render-html` is wired up in
  `HtmlContent.tsx` and in production use on product descriptions and FAQs. No new dependency.
  → §4
- **The content is small:** 178–2,069 words per page. → §2
- **The three T&C PDFs have no content record anywhere** — no title, version, effective date, not
  even a stored URL. They are S3 binaries whose links are hand-typed into a CMS HTML blob. Native
  rendering is impossible until someone re-authors them as content. These are also the
  commercially important documents (Sale, Hire, In-Stock Guarantee). → §1.2
- **The one real technical catch:** the CMS HTML is semantically flat. Headings are
  `<div class="contact-header">`, paragraphs are `<div class="text">`; `/refunds` contains zero
  `<p>`, `<ul>` or `<li>` tags. Each page also carries an embedded `<style>` block. So the
  existing renderer's tag styles won't fire — it needs `classesStyles` instead. Doable, but
  brittle because marketing hand-writes the HTML freehand. → §5.1
- **Recommended: a middle path.** Render the five CMS pages natively from the CMS (one source of
  truth, no forked copy), keep the three PDFs as links, and add a small document registry so
  those URLs stop being hardcoded in five places. → §6
- **Three things need fixing regardless of whether native pages happen** (all currently broken,
  all compliance-shaped): every "you agree to our Terms" link in the app is dead; signed-out
  users cannot reach legal content at all; and there is no privacy-policy URL in store metadata
  and no iOS privacy manifest. → §7.1
- **⚠️ Separately, and more urgent than anything above: the privacy policy is legally stale.** It
  cites the **Data Protection Act 1998** (repealed 2018), is dated "effective from 15/05/2012",
  and offers subject access "subject to the payment of a small fee (currently £10)... attach a
  cheque" — abolished by GDPR — while another paragraph in the same document claims full GDPR
  compliance. This is First Fence's to fix, not ours. Porting it natively would replicate it into
  a second channel. → §7.2

---

## 1. Where the legal content actually lives

### 1.1 The five web pages — a real CMS

One collection, one hand-authored HTML blob per page. Nothing legal is hardcoded in
`gatsby-website`.

```
Mongo cdn.cmspages (63 docs)
   -> cdn-graphql-v2   allCMSPages / cmsPage      [reads are UNAUTHENTICATED]
        -> gatsby-website  build-time source plugin -> one generic template
             -> dangerouslySetInnerHTML
```

| Hop | Evidence |
|---|---|
| Mongoose model | `cdn-graphql-v2/src/models/cms-page.js:3-17` |
| GraphQL type | `cdn-graphql-v2/src/schema/cms-page.js:5-20` |
| Queries registered | `cdn-graphql-v2/src/schema/index.js:127-129` |
| Resolvers — **no `authenticate()` on any query** | `cdn-graphql-v2/src/resolvers/cms-page.js:5-14` (only the 3 mutations are auth'd, at `:17,40,69`) |
| Gatsby source query | `gatsby-website/plugins/gatsby-source-firstfence/queries/cms-pages.js:1-16` |
| Page creation, one per CMS `path` | `gatsby-website/gatsby-node.js:449-454` (build filter `status in [PUBLISHED, TEST]` at `:104-110`) |
| The single template that renders all of them | `gatsby-website/src/templates/generic-pages/generic-pages.js:14-47` |
| CSS de-leaking shim | `gatsby-website/src/utils/scope-cms-html.js:1-40` |

There is **no `privacy-policy.js` or `legal-info.js`** in `gatsby-website/src/pages`.

Two API gotchas for whoever writes the mobile query:

- **There is no `cmsPageByPath`.** Unlike products (`productByPath` at `schema/index.js:84`), path
  lookup must go through the array-returning `allCMSPages(input: { path: "..." })`, which is an
  exact `.find(input)` match.
- **Do not use `allCMSPagesSearch` for path lookup.** It routes through `mongoFilter`, which turns
  every string into an *unanchored case-insensitive regex* (`src/utils/mongo-filter.js:83`) — so
  `"/privacy-policy"` also matches the archived `/privacy-policy-cookie-restriction-mode`.

Also note the page-builder fields (`blocks`, `builderData`, `useBuilder`, `renderedHtml`) exist in
the schema but are **dead in production**: `useBuilder: true` count is 0, `pagetemplates` is
empty, and every legal page is raw `html`. The Gatsby template never queries them. So there is no
structured/block representation to render from — only HTML.

### 1.2 The three T&C PDFs — no content record at all

These are plain S3 objects. They are live and actively maintained:

| Document | Size | Last modified |
|---|---|---|
| `docs/Terms and Conditions of Sale.pdf` | 290,723 B | **2026-08-06** |
| `docs/Terms-and-Conditions-of-Hire.pdf` | 344,538 B | 2026-06-02 |
| `docs/In+Stock+Guarantee+Terms+and+Conditions.pdf` | 327,061 B | 2025-12-01 |

Bucket is literally named `media.firstfence.co.uk`, fronted by CloudFront (dist `E21F3DNM33J1C5`),
versioning + SSE-AES256 on. Config: `website-api/config/s3-buckets.js:6-23`.

**Nothing anywhere stores these documents' title, version, effective date, or even their URL.**
The canonical copies of the three links are `<a href>` strings inside the **`/legal-info` CMS HTML
blob** — hand-typed, in three different encodings, one of which bypasses the CDN entirely:

```
.../docs/Terms%20and%20Conditions%20of%20Sale.pdf                       <- %20
.../docs/Terms-and-Conditions-of-Hire.pdf                               <- hyphens
https://s3.eu-west-1.amazonaws.com/media.firstfence.co.uk/docs/In+Stock+Guarantee+...pdf   <- raw S3, "+"
```

The same URLs are then re-hardcoded in **five** independent places, with **no single source of
truth** — change a filename and all five break silently:

1. Mongo CMS HTML (`/legal-info`)
2. `website-api` email templates — `resources/views/emails/order/order-summary.edge:35` and
   `orderRequest/order-summary.edge:35` (Hire); line `:111` of **four** separate footers
   (`emails/order/footer.edge`, `emails/orderRequest/footer.edge`, `emails/portal/orders/footer.edge`,
   `emails/portal/quotes/footer.edge`) for Sale
3. `gatsby-website` — `src/pages/payment-failed.js:60`, `src/pages/payment-cancelled.js:44`,
   `src/features/shopping-cart/summary/hire-deposit/hire-deposit.js:21`,
   `src/components/stock-guarantee-message/stock-guarantee-message.jsx:10`
4. `ff-uk-mobile` — `src/features/product/constants.ts:1-2`
5. A `customredirects` rule from 2025-12-22 that sends the legacy same-domain path
   `/docs/Terms%20and%20Conditions%20of%20Sale.pdf` to the **homepage** rather than the document

(`%20` and `+` both decode to spaces and resolve to the same object, which is why the
inconsistency has gone unnoticed.)

### 1.3 Who can edit what

| Asset | Who | Where | Guard rails |
|---|---|---|---|
| Policy HTML | any `admin` | `admin-website-v2/src/pages/content/cms-pages/edit-cms-page.jsx` — Monaco raw-HTML editor + iframe preview, save at `:167-194` → `updateCMSPage` mutation (`src/queries/cms-pages.js:56-63`) | history in `cmspagelogs`, **no approval step, no version or effective-date field** |
| Policy HTML | **an AI chat tool call** | `edit-cms-page.jsx:141-165` registers `mutators.setHtml`, dispatched by `src/hooks/use-client-tool-dispatcher.js:13-15` | none |
| The T&C PDFs | any active `admin`/`sales`/`depots` | `admin-website-v2` file manager → `website-api` `POST /admin/s3/upload` (`start/routes/admin.js:422-430`) | writes whitelisted to the `docs/` prefix (`S3ManagerController.js:104-109`); replacing = re-uploading the same key; audited in `s3_audit_logs` only since 2026-03-25 |

Two notes worth passing on: there is **no legal-specific admin screen at all** (a case-insensitive
grep of `admin-website-v2/src` for `privacy|terms|conditions|policy|legal|disclaimer|cookie`
returns zero hits — legal pages are just generic CMS rows), and the S3 manager's **read** paths
(`listObjects`, `searchObjects`, `downloadFile`) have no prefix restriction, so any admin can list
and download the whole media bucket.

---

## 2. The full legal inventory

From `cdn.cmspages` (production dump). Word counts matter because they set the effort ceiling.

| Path | Title | Status | Words | Reachable from |
|---|---|---|---|---|
| `/legal-info` | Legal Info (the hub) | PUBLISHED | 178 | web footer, checkout, mobile ("Terms & Conditions") |
| `/privacy-policy` | Privacy Policy | PUBLISHED | 2,069 | web footer, checkout, mobile |
| `/refunds` | Returns and Refunds Policy | PUBLISHED | 356 | `/legal-info` only |
| `/cookie-notice` | Cookie Notice | PUBLISHED | 579 | **mobile app only** — orphaned on the website |
| `/disclaimer` | Disclaimer | PUBLISHED | 1,121 | **nothing** — fully orphaned |
| `/terms-and-conditions` | Terms and Conditions | ARCHIVED | 1,269 | redirect → `/legal-info` |
| `/privacy-policy-cookie-restriction-mode` | Privacy and Cookie Policy | ARCHIVED | 1,242 | redirect → `/privacy-policy`; holds the **only real `<table>`** (17-row cookie table) in any legal page |

Plus the three PDFs (§1.2). No modern-slavery, accessibility, acceptable-use or GDPR-specific page
exists. `/sustainability-plan` is PUBLISHED and linked from the footer if it's wanted in the app.

**Two findings the RN team should know before wiring links:**

- **`/cookie-notice` is live but website-orphaned.** The mobile app is currently its *only* entry
  point (`LegalScreen.tsx:13`). It is not a 404 — but it is 579 words last touched 2025-08-29,
  with a hardcoded "Last updated: 28/07/2023" in its body.
- **Cookie policy is duplicated.** `/privacy-policy` contains its own "How we use cookies"
  section (which itself repeats one paragraph near-verbatim twice), and `/cookie-notice` covers
  the same ground again at 579 words. Two overlapping cookie policies, one of them invisible on
  the web.

---

## 3. What the mobile app does today

Everything is a WebView passthrough to the live website. There is no native legal content.

| Destination | How it opens now | Where |
|---|---|---|
| Privacy Policy → `firstfence.co.uk/privacy-policy` | in-app WebView | `src/features/account/screens/LegalScreen.tsx:11` → `:28` |
| Terms & Conditions → `/legal-info` | in-app WebView | `LegalScreen.tsx:12` |
| Cookie Policy → `/cookie-notice` | in-app WebView | `LegalScreen.tsx:13` |
| In-Stock Guarantee PDF | PDF handed to the WebView | `src/features/product/constants.ts:1-2`, used at `blocks/StockGuaranteeBlock.tsx:25-27` and `blocks/PriceBlock.tsx:60-62` |
| Product datasheet PDFs | external browser (`Linking.openURL`) | `blocks/DownloadsBlock.tsx:31` |

`LegalScreen.tsx` is 34 lines total: a header plus three menu items mapped from a hardcoded
`LEGAL_PAGES` array. Each renders an "external link" arrow icon (`:27`) while actually staying in
an in-app WebView. Note also the host inconsistency — `LegalScreen` uses bare `firstfence.co.uk`
while the rest of the app and the deep-link prefix use `www.firstfence.co.uk`.

**The In-Stock Guarantee PDF link needs verifying on Android.** Android's WebView cannot render
PDFs natively, so handing it a PDF URL typically shows nothing or triggers a download.

---

## 4. Is it possible? Yes — for the CMS pages, with no backend work

Three things line up:

1. **The API is already open.** CMS reads need no auth (`resolvers/cms-page.js:5-14`), CORS is
   bare `cors()` (`cdn-graphql-v2/server.js:110`), and the mobile app's GraphQL client already
   sends no auth headers (`ff-uk-mobile/src/store/api/graphqlApi.ts:7,25`) against this same API.
2. **The renderer already exists and is battle-tested.** `react-native-render-html@6.3.4` is
   wrapped by `src/features/product/components/HtmlContent.tsx` — brand fonts (`:10-16`),
   `tagsStyles` for `p/h2/h3/h4/strong/b/a/ul/ol/li` (`:25-49`), link interception routing to
   WebView or `mailto:`/`tel:` (`:65-78`), junk-tag stripping (`:52`). In production use by
   `DescriptionBlock.tsx:35` and `FaqBlock.tsx:39`.
3. **The route and deep link already exist.** `Legal` is in `src/navigation/types.ts:57`,
   registered at `src/navigation/RootNavigator.tsx:208`, and `linking.ts:43` already maps
   `Legal: 'legal'` — so `ffuk://legal` resolves today.

So the mechanical work is roughly: add an RTK Query endpoint for

```graphql
query LegalPage($path: String!) {
  allCMSPages(input: { path: $path, status: PUBLISHED }) {
    path title html updated
  }
}
```

take `[0]`, feed `html` into `HtmlContent`, add one route + one `linking.ts` entry per document.
No new dependency, no backend change.

---

## 5. Is it worth it? Mostly yes — with two honest caveats

### 5.1 The HTML is semantically flat, so `HtmlContent` won't style it as-is

This is the one genuine technical obstacle. The CMS HTML doesn't use semantic tags:

| What it looks like | What it actually is |
|---|---|
| `<div class="title">` | page banner |
| `<div class="contact-header">` | a section heading (13 of them in `/privacy-policy`) |
| `<div class="text">` | a paragraph |
| `<div class="list">` | a list wrapper |

`/refunds` contains **zero** `<p>`, `<ul>` or `<li>` tags — its whole body is 10 `div.text` blocks.
`HtmlContent`'s `tagsStyles` keys off tag names, so those divs would render unstyled.

Each page also opens with an embedded `<style>` block (667–1,291 chars) full of generic global
class names and desktop `@media (max-width: 800px)` breakpoints. Both existing consumers already
need a shim for this (`gatsby-website/src/utils/scope-cms-html.js`, and a twin in
`admin-website-v2/src/utils/`).

**The fix is small:** `react-native-render-html` supports `classesStyles`, so map
`.title/.contact-header/.text/.list` to the app's `AppText` scale, and add `style` to
`IGNORED_DOM_TAGS` (`HtmlContent.tsx:52`) so the desktop CSS is dropped rather than half-applied.

**The residual risk is real though:** marketing hand-writes this HTML in a Monaco editor with no
schema. A class-name-based renderer breaks silently the moment someone types `<div class="Text">`
or pastes from Word. `/legal-info` already contains malformed markup — an `<li>` that is never
closed before the next opens. Browsers recover; a strict RN parser may not. Mitigate with a
fallback: if parsing yields nothing, fall back to the WebView.

### 5.2 The three T&C PDFs cannot be made native

Sale, Hire and In-Stock Guarantee are 290–345 KB PDFs with no content record. Making them native
means someone re-authoring three contractual documents as CMS pages — a legal review exercise, not
an engineering task. Adding a PDF viewer dependency instead just reproduces what the WebView does.

So "native legal pages" will be **partial by necessity**: five native pages plus three PDF links.
Worth being explicit with the client about that, because a Legal screen where three of eight items
behave differently is a design question.

### 5.3 The argument in favour

- **It removes a fork risk rather than creating one.** Rendering from the CMS keeps one source of
  truth. The alternative some teams reach for — pasting legal copy into the app bundle — means the
  app silently diverges from the website the next time marketing edits a policy.
- **The volume is trivial** (§2) and there is no translation burden: the app has no i18n framework
  at all, and for a UK-only V1 it needs none.
- **It fixes a genuinely poor experience.** Today the app renders desktop-CSS pages in a WebView.
- **Offline and error states become possible.** A WebView shows a browser error page; a native
  page can cache and use the app's `AlertBox`.
- The app already has the primitives for long-form documents: `AppText` (`h1`–`h6` +
  `bodyXl`–`bodyXs` × 4 weights, `src/ui/components/AppText.tsx:5-39`), `Section.tsx` (an
  accordion explicitly built to replace web tabs on mobile), `ScreenHeader`, `Divider`, `AlertBox`.

---

## 6. Recommendation — three tiers

### Tier 1 — fix what's broken (do first, independent of native pages)

These are compliance-shaped and cheap. See §7.1 for the evidence.

1. Wire up the four dead "you agree to our Terms" links.
2. Move `Legal` and `WebView` out of the authenticated-only branch of `RootNavigator`.
3. Add a privacy-policy URL to store metadata; add an iOS privacy manifest.
4. Verify the In-Stock Guarantee PDF actually displays on Android.

### Tier 2 — native rendering for the five CMS pages (recommended)

Query `allCMSPages` by path → render with the existing `HtmlContent` plus a `classesStyles` map
(§5.1) → one route and one `linking.ts` entry per document → WebView fallback on parse failure.
Keep the three T&C PDFs as links. Decide whether to surface `/disclaimer` (currently orphaned
everywhere) and `/sustainability-plan`.

### Tier 3 — a document registry for the PDFs (propose to the client)

Add a small `LegalDocument` type to the CMS — `slug`, `title`, `kind`, `url`, `version`,
`effectiveDate`, `fileSize` — so:

- the app resolves a URL instead of hardcoding one, killing the five-copies problem (§1.2);
- replacing a T&C becomes a content operation with a version and a date, not an S3 key overwrite;
- a future "policy changed, please re-accept" flow has something to diff against. Note there is
  **no `lastUpdated` field** on CMS pages either — `updated` changes on any edit, including a typo
  fix, and the human-readable date is buried in prose.

Also worth adding a `cmsPageByPath` query to match the existing `productByPath`.

---

## 7. Problems found along the way

### 7.1 In the app — all four T&C acceptance points are dead links

The app tells users they have agreed to terms they have no way to read.

| Flow | The copy | What actually happens | Where |
|---|---|---|---|
| Register | "By signing in, you agree to our **Terms** and **Privacy Policy**" | `onPress={() => {}}` on **both** | `src/features/auth/screens/RegisterScreen.tsx:205-214` (no-ops at `:207`, `:211`) |
| Checkout | "By continuing, you agree to the **Terms & Conditions.**" | `Alert.alert('Coming soon', ...)` | `src/features/checkout/screens/CheckoutScreen.tsx:332-341` (`soon` at `:117`) |
| Payment methods (PayPal) | "…you agree to our **Terms and Conditions**" | **no `onPress` at all** — styled blue, not tappable | `src/features/checkout/screens/PaymentMethodsScreen.tsx:232-239` |
| Pay for quote | "By continuing, you agree to the **Terms & Conditions.**" | **no `onPress`** — not tappable | `src/features/quotes/screens/PayForQuoteScreen.tsx:184-189` |

There is no acceptance checkbox anywhere — all four are passive "by continuing" copy.

**And even wiring them up isn't enough:** `Legal` and `WebView` are registered **only** inside the
`token ?` branch of `RootNavigator.tsx:70-219`. The signed-out branch (`:220-238`) has only
`SignIn`, `Register`, `ForgotPassword`, `ResetPassword`, and `bootstrapAuth` issues no guest token
(`src/features/auth/store/bootstrapAuth.ts:11-14`). So a signed-out user **cannot reach legal
content at all**, and `navigate('WebView', …)` from the Register screen would fail outright.

**Store readiness gaps:** no privacy-policy URL anywhere in `app.config.ts` or `eas.json`, and no
`ios.privacyManifests` / `PrivacyInfo.xcprivacy` (Apple requires a privacy manifest declaring
required-reason APIs). Both stores need a reachable privacy-policy URL in the listing, and Apple
wants in-app access to terms wherever an account is created or a purchase made.

Also dead: the `Support` menu item, `onPress={() => {}}` at
`src/features/account/screens/ProfileScreen.tsx:133`.

### 7.2 ⚠️ In the content — the privacy policy is legally stale

**This outranks the whole native-pages question and belongs with First Fence, not the dev team.**
`/privacy-policy` currently:

- cites the **Data Protection Act 1998**, repealed in 2018;
- states "This policy is effective from **15/05/2012**";
- offers subject access "subject to the payment of a small fee (currently **£10**)... attach a
  cheque" — a charge and a mechanism abolished by GDPR;
- and, in the same document, claims "First Fence is 100% compliant with the General Data
  Protection Regulation (GDPR)".

Related content defects: `/legal-info` reads "This site is © **2012** First Fence Limited" (while
the web footer computes the year dynamically at
`gatsby-website/src/layout/footer/terms-and-conditions/terms-and-conditions.js:7-13`); the
company address is inconsistent between pages (`/legal-info` says "Kiln Way, Swadlincote,
Derbyshire"; `/refunds` and `/privacy-policy` say "Kiln Way, **Woodville**, Swadlincote,
Derbyshire"); and `/cookie-notice` carries a hardcoded "Last updated: 28/07/2023".

**Recommendation: raise this with First Fence before building anything.** Porting the text
natively would replicate a non-compliant document into a second customer-facing channel, and a
native page makes it *more* prominent, not less.

### 7.3 Governance

Legal text is editable by any admin — and by an AI chat tool call (§1.3) — with no approval step
and no version or effective-date field. PDF replacement is a same-key S3 upload, audited only
since March 2026; before that, staff used a desktop Electron S3 client outside any audited
workflow (per the commit history around `5a301e2`, *"re-add ACL checking matching Electron app
approach"*). If the app is going to assert "you agreed to these terms", a version field and an
approval step are worth proposing.

### 7.4 Minor / latent

- **`AWS_REGION` is never set.** `cdn-graphql-v2/src/services/s3.js:10` reads
  `process.env.AWS_REGION`, but the env files only define `AWS_BUCKET_REGION`
  (`.env.examle:15` — note the filename typo). The S3 client gets `region: undefined`.
- **Region disagreement for the same bucket:** `website-api/config/s3-buckets.js:9` says
  `eu-west-2`; `website-api/.env.example:52` says `https://s3.eu-west-1.amazonaws.com`, and the
  raw-S3 URL baked into `/legal-info` uses `eu-west-1`.
- **The CMS template doesn't filter by status.** `generic-pages.js:36` matches on `path` alone;
  only `gatsby-node.js:104` filters `PUBLISHED`/`TEST`. Harmless today.
- **`cmspages` documents carry a `deleted` field that isn't in the Mongoose schema**, so it can't
  be filtered through GraphQL. All legal pages are `deleted: false` today.
- `cmspagelogs` preserves a typo'd filename variant
  (`In+Stock+Guarantee+Terms+and+Conditions++.pdf`) — evidence of manual filename fiddling on a
  live legal document link.
- `expo-file-system` and `expo-sharing` are installed in the app but referenced nowhere in `src/`.
  `expo-web-browser` is installed but used only for the Worldpay redirect
  (`src/features/checkout/utils/redirectPayment.ts:1,35`).
- The `docs/` S3 prefix is manually curated — ~80 distinct URLs with three inconsistent
  encodings, ad-hoc subfolders (`docs/ISO/`, `docs/assembly-guides/…`,
  `docs/account-application-forms/`), one doubled extension
  (`April%202026%20-%20Locinox.pdf.pdf`), and no manifest or index object.

---

## 8. Open questions for the client / product

1. Should the three T&C documents (Sale, Hire, In-Stock Guarantee) be **re-authored as CMS pages**
   so they can be native, or do they stay PDFs indefinitely? This is the single biggest scoping
   question — and it's a legal-review exercise, not a dev one.
2. Is a **mixed Legal screen** acceptable for V1 — five native pages plus three PDF links?
3. Does the **privacy policy get updated before or after** the app ships legal pages? (§7.2)
4. Should `/disclaimer` (orphaned everywhere) and `/cookie-notice` (orphaned on the website)
   appear in the app, and should the duplicated cookie content be consolidated?
5. Do we want a **version + effective-date field** on legal documents, to enable "terms changed,
   please re-accept"? Nothing supports that today.
6. Should legal edits require **approval**, given any admin (or an AI tool call) can rewrite them?

---

## 9. Code reference index

**cdn-graphql-v2** (the CMS API)
- `src/models/cms-page.js:3-17`, `src/schema/cms-page.js:5-20` — model + type
- `src/schema/index.js:127-129` — `allCMSPages`, `allCMSPagesSearch`, `cmsPage`
- `src/resolvers/cms-page.js:5-14` — unauthenticated reads
- `src/utils/mongo-filter.js:83` — the unanchored-regex gotcha
- `server.js:92,110,114-120` — introspection on, bare CORS, non-rejecting auth context
- `src/services/s3.js:10` — the `AWS_REGION` bug

**gatsby-website**
- `gatsby-config.js:66-71` — `gatsby-source-firstfence` is the only content source plugin
- `plugins/gatsby-source-firstfence/queries/cms-pages.js:1-16`, `.../nodes/cms-page.js:3-24`
- `gatsby-node.js:104-110`, `:449-454` — build filter + page creation
- `src/templates/generic-pages/generic-pages.js:14-47` — the single template
- `src/utils/scope-cms-html.js:1-40` — `<style>` stripping + selector rescoping
- PDF links: `src/pages/payment-failed.js:60`, `src/pages/payment-cancelled.js:44`,
  `src/features/shopping-cart/summary/hire-deposit/hire-deposit.js:21`,
  `src/components/stock-guarantee-message/stock-guarantee-message.jsx:10`
- Page links: `src/layout/footer/terms-and-conditions/terms-and-conditions.js:33,39`,
  `src/features/checkout/contact-details-page/terms-and-conditions/terms-and-conditions.js:26`,
  `src/features/checkout/ui/privacy-note/privacy-note.js:22`

**ff-uk-mobile**
- `src/features/account/screens/LegalScreen.tsx:10-14,27,28` — the three WebView links
- `src/features/product/constants.ts:1-2` — hardcoded guarantee PDF URL
- `src/features/product/components/HtmlContent.tsx:10-78` — the existing HTML→native renderer
- `src/features/browser/screens/WebViewScreen.tsx:10-40` — WebView wrapper with fallback
- `src/navigation/types.ts:57,59`, `RootNavigator.tsx:70-219,208,214-218`, `linking.ts:7,43,45`
- `src/ui/components/AppText.tsx:5-39`, `Section.tsx:15` — long-form primitives
- Dead legal links: `RegisterScreen.tsx:205-214`, `CheckoutScreen.tsx:332-341,117`,
  `PaymentMethodsScreen.tsx:232-239`, `PayForQuoteScreen.tsx:184-189`, `ProfileScreen.tsx:133`
- `app.config.ts` — no privacy URL, no `ios.privacyManifests`

**website-api**
- `config/s3-buckets.js:6-23` — bucket config, `allowedPrefixes: ["docs/"]`, `fullAccessUserIds`
- `start/routes/admin.js:422-430` — the S3 manager routes
- `app/Controllers/Http/S3ManagerController.js:104-109` (write gate), `:449-457` (upload ACL),
  `:481-484` (CloudFront invalidation), `:485-519` (audit)
- Email PDF links: `resources/views/emails/order/order-summary.edge:35`,
  `orderRequest/order-summary.edge:35`, and `:111` of the four `footer.edge` files
- `database/migrations/1715855664102_setting_schema.js:8-14` — the `settings` table (holds nothing legal)

**admin-website-v2**
- `src/pages/content/cms-pages/edit-cms-page.jsx:141-165,167-194,263-335` — the raw-HTML editor,
  save path, and AI `setHtml` mutator
- `src/atoms/html-editor/html-editor.jsx:19-60,214-229` — Monaco + HTMLHint
- `src/queries/cms-pages.js:56-63` — `updateCMSPage`
- `src/pages/file-manager/s3-manager-page.jsx:78,116-122`,
  `components/s3-upload-modal.jsx:19,136-142` — PDF replacement UI
- `src/layout/routes/data/content.js:260-287`, `data/pages.js:204-210` — admin-only routing

**Data** — Mongo `cdn.cmspages` (63 docs), `cmspagelogs` (62), `customredirects` (1,464),
`pagetemplates` (0). MySQL `website`: `settings`, `uploads`, `s3_audit_logs` all 0 rows.
