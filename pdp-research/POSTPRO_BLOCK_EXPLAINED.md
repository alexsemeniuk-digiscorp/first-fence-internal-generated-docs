# PostPro calculation service — what it is, where it lives, and what the upload is for

A follow-up to block 19 of `PRODUCT_PAGE_STRUCTURE.md` and block L of `PRODUCT_PAGE_STRUCTURE_SCOPED.md`.
Neither of those files has been changed.

**Why this exists.** The visual mockup at `first-fence-pdp-components.vercel.app` renders this block as a
checkbox plus a button reading **"Add to basket and upload"**, and the designer could not find anything like
it on the live site.

**Verdict: the block is real and live, but the mockup's button is wrong, and the upload is not on the
product page at all.** Details below, all verified against the source, the production database, and live
production HTML.

---

## 1. The three things that went wrong

| | Mockup / doc says | Reality |
|---|---|---|
| Button label | "Add to basket and upload" | **"Add to basket"** — the words "and upload" do not exist anywhere in the codebase |
| Where the upload is | implied on the product page | **Inside a modal** that opens *after* pressing the button. There is no file input on the product page |
| Price source | "hard-coded … **not** from data" | The *label* is hard-coded; the price actually charged **is** from the database. Both are £1,368.75 today |

Searching the live site for "and upload" would find nothing — which is exactly what happened.

---

## 2. It is live — on 63 products

The block renders only when the product's `showPostPro` field is on.

| | Count |
|---|---|
| Products with `showPostPro: true` (any status) | 92 |
| …that are published, visible, and get a real page | **63** |

**Verified live example:** `https://firstfence.co.uk/1-8m-echogroove-reflective-acoustic-fencing-kit`

The rendered block on that page, in order:

```
Use Our PostPro Calculation Service                                    ← h2
Unsure if our recommended steel posts are right for you? Simply tell
us about your site and height for an engineers report on post
suitability:                                                           ← paragraph

☐  I have all my site details and require Permanent Works PostPro
   Calculations                                     £1,368.75          ← one checkbox

                                          [ Add to basket ]            ← button
```

That is the whole block. Four elements. **One checkbox, one button, no file input, no upload affordance.**

All 63 are **acoustic fencing kits** — EchoGroove, EchoReflect, EchoAbsorb ranges, mostly `CALCULATOR` type.
That is the other half of why it was hard to find: it is 63 products out of ~3,500, all in one niche
category, and it sits low on the page below the configurator.

---

## 3. What PostPro actually is

PostPro is a **paid engineering report**, sold as a product in its own right, bolted onto the fencing
product page.

Acoustic fencing panels are tall and heavy, so the steel posts holding them have to be sized for the
specific site — ground conditions, wind loading, height, whether the structure is permanent or temporary.
First Fence's suggested posts are generic. If the customer needs a signed-off engineer's report saying
*"these posts are correct for your site"*, they buy PostPro.

The modal's own subheading states it plainly:

> "Looking for an approved Engineers report for your site specific fencing posts?"

The terms text is equally explicit that it is **third-party**: First Fence do not produce the report, do not
guarantee it, and disclaim liability for it. That framing matters — this is a referral product, not a
First Fence service.

---

## 4. So what is the upload for?

**Engineers cannot produce a site-specific report without site documents.** The modal exists to collect
them, along with the structured facts about the site, at the moment of purchase — so the order arrives
complete instead of triggering an email chain.

Three document types are collected, each behind a yes/no question:

| Question | Uploads if "yes" |
|---|---|
| "Do you have Ground Investigation (GI) Data?" | soil / borehole survey files |
| "Do you have a site plan, which includes any neighbouring buildings, utilities, roads or infrastructure?" | site plan |
| "Do you have a fencing arrangement plan of your proposed boundary, including any openings and gaps?" | fencing layout plan |

Answering "no" is allowed for all three — the upload area simply doesn't appear. So **the upload is
optional in every case**, which is worth knowing: the flow must work end-to-end with zero files attached.

### Upload rules

| | |
|---|---|
| Accepted types | PNG, JPEG, PDF only |
| Max size | 10 MB per file |
| Multiple files | Yes, per category |
| Rejected files | Listed by name under "The following files are not allowed:" |
| Preview | Image thumbnail, or a generic PDF icon for PDFs, with the filename and an "X" to remove |

Validated twice — in the browser and again server-side.

### What happens to the files

Files go to **S3** under `uploads/post-pro/`, renamed to a UUID, and an `Upload` row links each one to the
cart, the PostPro product, and its category. The structured answers are written to a separate SQL table
`post_pro` (address, coordinates, three-words, type, height, temp-or-perm, additional text). That is how the
warehouse and the engineers pick the job up.

---

## 5. The modal, screen by screen

Opening the modal is a **two-step** action: tick the checkbox, then press "Add to basket". The modal is the
real form; the product-page block is just its entry point.

**1 — Location.** Address lookup, which resolves a postcode to coordinates, which drops a marker on a
Google Map (50vh tall) that the user can drag to fine-tune the exact spot. Latitude and longitude are also
editable as raw number fields. There are three what3words fields, but their validation is commented out, so
they are optional in practice.

**2 — "Which system do you require?"** Three large image cards, pick one: **Reflective panel**,
**Groovy panel**, **Absorbent panel**.

**3 — Panel height.** A dropdown: 1.8 / 2.0 / 2.4 / 2.5 / 2.7 / 3.0 / 3.5 / 4.0 / 4.2 / 5.0 m.

**4 — Works type.** Dropdown: Permanent or Temporary.

**5, 6, 7 — The three document questions** from section 4. Each is a yes/no dropdown that reveals a file
picker when set to "yes".

**8 — Other details.** Free-text area, **255 characters max**.

**9 — Terms and conditions.** Two paragraphs of third-party disclaimer plus a mandatory
"I accept the terms and conditions" checkbox.

**10 — Captcha.** Present in the code but **commented out**. There is currently no bot protection on this
form.

**Footer** — "Need help with PostPro? Speak to our team on 01283 512 111", then **Add To Basket** and
**Cancel**.

### Modal states

| State | What shows |
|---|---|
| Loading | A single centred spinner replacing the entire body |
| Normal | The form above |
| Validation failed | Red border on the offending section, a message, plus "Please correct the errors above" at the bottom |
| Error | "Sorry there was an error. Please try again later." and a single **Close** button |
| Success | "Item added to basket", with **Continue Shopping** and **View Basket** buttons |

Note the success state **replaces the whole form** — there is no return path to edit what was just
submitted.

---

## 6. Price: correcting the original doc

The original doc said the price is "hard-coded … **not** from data". That is half right, and the half that
is wrong matters.

- **The £1,368.75 on the checkbox label is hard-coded** in the page source. Confirmed.
- **The price actually charged is not.** The modal fetches a hidden Mongo product on open —
  `66a0cc683cd5eb05192dddc6`, titled "Post Pro Calculations", `visible: false`, path `/post-pro-calculations`
  — and that product's database price is **£1,368.75**.

So the two agree today. But they are two separate values, and **nothing keeps them in sync** — an admin
editing the product price would change what the customer is charged while the page carries on advertising
the old number. Worth raising with the team regardless of what the mobile app does.

---

## 7. The second option that no longer exists

The source contains a second, commented-out checkbox:

> "I need a site visit, survey and Permanent Works PostPro calculations — **£1,992.30**"

Its modal (`SiteVisitModal`) is still mounted on every page that renders the block, and the click handler
still branches to it — but with the checkbox commented out, nothing can ever reach it. Its product also
still exists in the database: `/post-pro-with-site-visit`, "Post Pro Site visit", **£1,849.99**, hidden.

Note the commented label says £1,992.30 while the product says £1,849.99 — **stale even before it was
disabled.** Good evidence for treating any price in the page source as untrustworthy.

**Consequence for the design:** the block is a checkbox group with exactly one checkbox. The checkbox
carries no information — you must tick the only option before the only button works. If the site-visit
option is not coming back, this should be a single button, not a checkbox plus a button.

---

## 8. Four bugs in the live block

Flagging these so the app doesn't inherit them.

**1 — Nothing ticked, button does nothing, no feedback.** The handler matches the ticked option against two
known strings; with nothing ticked it matches neither and returns silently. No message, no disabled state,
no visual change. The button looks broken.

**2 — Two validation messages can never appear.** The validator writes its errors under the keys
`sitePlanFiles` and `layoutPlanFiles`, but the form reads `sitePlan` and `fencingLayout`. So if the user
answers "yes" to *Do you have a site plan?* or *Do you have a fencing layout?* and then uploads nothing,
submission is blocked, the generic "Please correct the errors above" appears at the bottom — and **the
section causing it says nothing at all.** The Ground Investigation section is wired correctly, so users get
a message there and silence for the other two. This is a genuine dead end and the most likely support call
in the whole flow.

**3 — A £749.99 price is copy-pasted onto every answer.** All six structured answers (address, coordinates,
panel type, height, temp/perm, notes) are sent to the basket as priced options at **£749.99 each**, flagged
`oneOff`. That is the Temporary Works Design product's price, pasted six times.

The **amount charged is not affected** — the line total is built from the product's own extra-price field,
which is zero here. But the order-email builder routes every `oneOff` option into an "Added Extras" list
*with its price*, and the email template prints that price. So an order confirmation is likely to list six
extras at £749.99 (£4,499.94 of phantom line items) against a correctly-charged £1,368.75. **Verify against
a real order email before acting** — but the code path is unambiguous, and the app must not copy the
pattern.

**4 — The label isn't attached to its checkbox.** The label's `htmlFor` says `calculationsUpload`; the
input's id is `calculationsUploadInput`. Tapping the text only works because the surrounding container has
its own click handler. On a phone this is exactly the kind of thing that produces a dead tap target near
the edges.

---

## 9. What this means for the mobile app

1. **Fix the mockup first.** Button reads **"Add to basket"**. No upload control on the product page.
2. **This is a full-screen flow, not a modal.** The web crams a ten-section form — including a 50vh Google
   Map and three file pickers — into a Bootstrap modal declared `size="sm"`. On a phone that is not
   survivable. Make it a pushed screen, ideally a multi-step wizard following the ten sections in section 5.
3. **A native file picker is a different problem from a web one.** On mobile the sources are the camera
   roll, the camera itself, and Files/Drive. A site plan is usually a PDF that lives in email or cloud
   storage, not the camera roll — the picker must reach a document provider, not just photos.
4. **Design the zero-file path properly.** All three uploads are optional. The most common completion is
   probably "no, no, no" plus a note — that path should be fast, not buried under three collapsed upload
   areas.
5. **Take the price from the API**, never from a string in the app. Section 6 explains why.
6. **Decide whether it ships at all.** 63 products, one product family, a third-party service the business
   explicitly disclaims. It is a defensible thing to cut from v1 and replace with "call us" — but if it
   ships, it needs the whole ten-section form, because a half-collected PostPro order is useless to the
   engineers.
7. **Give the button a disabled state** and wire the two dead validation messages.

---

## 10. Source references

**Frontend** (`gatsby-website`):
- `src/features/post-pro/post-pro.js` — the product-page block; the £1,368.75 label, the commented-out
  second checkbox, and the silent no-op handler
- `src/features/post-pro/calculations-upload-modal/calculations-upload-modal.js` — the whole modal; the
  hidden product id at the top, the basket-then-upload sequence in `addToBasket`
- `src/features/post-pro/calculations-upload-modal/upload-files.js` — PNG/JPEG/PDF, 10 MB, previews,
  rejected-file list
- `src/features/post-pro/calculations-upload-modal/validateForm.js` — the two mismatched error keys
- `src/features/post-pro/calculations-upload-modal/processOptions.js` — the six £749.99 options
- `src/features/post-pro/location-section.js` — address lookup, Google Map, draggable marker
- `src/features/post-pro/site-visit-modal/site-visit-modal.js` — mounted but unreachable
- `src/templates/product/product.js:484` and `src/templates/group-product/group-product.js:345` — the
  `showPostPro` gate

**Backend** (`website-api`):
- `start/routes/public.js:105` — `POST /upload/post-pro`
- `app/Controllers/Http/FileUploadController.js:304` — S3 upload, `Upload` rows, `post_pro` row
- `database/migrations/1721654466010_post_pro_schema.js` — the `post_pro` table
- `app/Utils/order-helpers.js:26` and `app/Utils/price-calculations.js:34` — `oneOff` options become priced
  "extras"
- `resources/views/emails/order/products.edge:98` — the "Added Extras" block that prints their prices

**Data** (Mongo `cdn`):
- `66a0cc683cd5eb05192dddc6` — "Post Pro Calculations", £1,368.75, hidden
- `669926e63cd5eb051952b631` — "Post Pro Site visit", £1,849.99, hidden
- `products.showPostPro` — the on/off switch, true on 63 live products
