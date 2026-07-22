# "Available with lead time" — what it is and when it appears

A plain-language companion to `PRODUCT_PAGE_STRUCTURE.md`. Those docs mention this badge in one line and
assume you already know how lead times work. This one explains it from scratch. Nothing else is changed.

---

## 1. First, what "lead time" means here

First Fence sells two kinds of stock:

- **In stock** — it's in the warehouse, it ships on the normal schedule.
- **On a lead time** — it isn't in the warehouse. It has to be made or delivered in first. The shopper can
  still order it today; it just takes longer to arrive.

"Lead time" is that waiting period, always counted in **working days** (Saturdays and Sundays are skipped).

So every product is showing the shopper one of two messages, all the time:

> ☑ **In Stock**  or  ☑ **10 Working Days Lead Time**

"Available with lead time" is a **third** message — and it exists for one specific reason, explained in
section 4.

---

## 2. Where lead time shows up on a product page

Three separate places, and they do **not** always agree with each other. This is the single most important
thing to understand.

| # | Where | What it looks like |
|---|---|---|
| 1 | **Chip on the product image**, top-left corner | Small solid-colour rounded rectangle. Green "In Stock", or **grey** with the lead-time text |
| 2 | **Text line under the price** | A small tick-box icon + text. Green "In Stock", or **orange** with the lead-time text |
| 3 | **Product cards** in "Related Products" / category lists | Small text label at the top of the card |

They are three different components reading the same data with slightly different rules. They can disagree.

### Exact styling of each

**The image chip** (`badges.jsx`) — absolutely positioned 16px from the top and 16px from the left of the
image, chips laid out in a row with an 8px gap. Each chip: 10px padding, 4px corner radius, 14px text,
weight 600, white text.

| Chip | Background |
|---|---|
| In Stock | green `#2ba471` |
| *lead time (any wording)* | **grey `#969696`** |
| Best Seller | blue `#4e9bd1` |
| Sale | red `#ff0000` |
| Buy More Pay Less | cyan `#0099cc` |
| New | green `#008000` |

The stock chip is always first; the others follow in that fixed order when their flags are on.

**The text line under the price** (`stock-message.js`) — an 18px tick-box icon (12px on mobile) with a 6px
gap, weight 500 text, 10px vertical padding (3px bottom only on mobile, 14px text).

| State | Colour |
|---|---|
| In Stock | green `#00ad07` |
| lead time | **orange `#d47300`** |

Note two oddities worth fixing rather than copying:

- **The chip is grey but the text line is orange** for the exact same state. Two different colours for one
  concept, on one screen.
- **Both states use the same tick-box icon.** A tick box next to "10 Working Days Lead Time" reads like a
  confirmation, not a delay.

---

## 3. Every possible wording

There is exactly **one** function that produces every lead-time string on the site
(`getLeadTimeMessage` in `src/utils/product/lead-time.js`). It takes a number of working days and returns
text. That's the whole vocabulary:

| Number of working days | Text shown |
|---|---|
| `0`, empty, or missing | **"Available with lead time"** |
| 1 | "1 Working Day Lead Time" |
| 2, 3, 4 | "2 Working Days Lead Time" … |
| 5 | "1 Week Lead Time" |
| 7 | "7 Working Days Lead Time" |
| 10 | "2 Weeks Lead Time" |
| 13 | "13 Working Days Lead Time" |
| 15 | "3 Weeks Lead Time" |
| 20, 25, 30 … | "4 Weeks", "5 Weeks", "6 Weeks" Lead Time |

**The rule:** if the number divides evenly by 5 it is shown as **weeks**; otherwise it is shown as
**working days**. There is never a mixed "1 Week 2 Days" form.

That produces a jump the designer should know about: **5 → "1 Week", but 6 → "6 Working Days".** Going up
by one day makes the label get *longer*, not shorter. Same at 10 → 11, 15 → 16, and so on.

**Longest string you must fit:** "13 Working Days Lead Time" is live today, and values up to 60 exist, so
plan the chip and the text line for roughly **26 characters**. On a narrow phone the chip will need to wrap
or the image will need more headroom — the web version never wraps because it has the width to spare.

### A real bug in the wording

For 6, 11, 16, 21 … days the code picks the **singular** word but prints the full number:

> "6 Working **Day** Lead Time"

It happens because the code checks `duration % 5 === 1` (which is true for 6, 11, 16 …) and treats that as
"one day". No live product currently uses those values, so nobody has noticed. **Don't reproduce it in the
app** — just pluralise on the actual number.

---

## 4. So what actually triggers "Available with lead time"?

Two different fields describe lead time, and they can contradict each other:

| Field | Type | Meaning |
|---|---|---|
| `leadTime` | true / false | "This product has a lead time." A yes-or-no flag. |
| `leadTimeDuration` | a number | "The lead time is *this many* working days." |

**"Available with lead time" is what the page says when the flag says yes but the number is missing.**

It is a fallback for incomplete data. It means: *we know this isn't in stock, but nobody filled in how long
it takes.* That's it. It is not a special product category, not a marketing message, and not a state anyone
designed on purpose.

### The four combinations

| `leadTime` flag | `leadTimeDuration` | Image chip | Text line under price |
|---|---|---|---|
| off | 0 / empty | green **"In Stock"** | green **"In Stock"** |
| off | 10 | grey **"2 Weeks Lead Time"** | orange **"2 Weeks Lead Time"** |
| **on** | **0 / empty** | grey **"Available with lead time"** | green **"In Stock"** ← they disagree |
| on | 10 | grey **"2 Weeks Lead Time"** | orange **"2 Weeks Lead Time"** |

**Row three is the whole story.** The chip and the text line ask different questions:

- The **chip** asks *"is either the flag on, or is there a number?"* → the flag alone is enough.
- The **text line** asks *"is the number greater than zero?"* → the flag alone is ignored.

So when a product has the flag on but no number, the image says "Available with lead time" while the line
right under the price says "In Stock", on the same screen, at the same moment.

**Consequence:** the text line under the price can **never** show "Available with lead time". It only ever
prints a real duration or "In Stock". The wording lives exclusively on the **image chip** and on
**product cards**.

---

## 5. How the number is worked out

`leadTimeDuration` is not simply read off the product. It is recalculated on every interaction, in three
steps:

1. **Start with the product's own `leadTimeDuration`.**
2. **Take the longest lead time among the options the shopper has selected.** If any selected option has its
   own lead time and it's longer than what we have so far, that becomes the number. *(Longest wins — not the
   sum, not the last one picked.)*
3. **Add working days until `dateDueBackInStock`,** if that date is set and is in the future. This is a
   count of working days from today to that date, including today, skipping weekends. If the date is in the
   past it is ignored.

**Step 2 is the one that matters most for design.** It means the stock state is **not fixed for a product** —
it changes as the shopper configures it.

Concrete live example — a "Finish" swatch group:

| Option | Its own lead time |
|---|---|
| Galvanised | none |
| GREEN – RAL 6005 | none |
| BLACK – RAL 9005 | none |
| BLUE – RAL 5010 | 5 days |
| RED – RAL 3020 | 5 days |
| GREY – RAL 7037 | 5 days |
| YELLOW – RAL 1023 | 5 days |

The product itself has no lead time. So it loads as **"In Stock"** — and the moment the shopper taps the
blue swatch it becomes **"1 Week Lead Time"**, in the chip *and* the text line, at once. Tapping back to
Galvanised returns it to "In Stock".

**This is common, not an edge case:** 1,259 of 4,500 option groups contain at least one option carrying a
lead time, and **727 live products start as "In Stock" and can be flipped to a lead time by picking an
option.**

For the mobile design that means:

- The stock indicator must be **live**, not a static badge painted once when the screen opens.
- It sits above the variant pickers on the web, so the thing that changes is off-screen from the control
  that changed it. On a phone that is worse, not better. **Consider putting a stock line near the variant
  picker, or animating the change so it isn't silent.**
- Step 3 means the number can change **between sessions** with no data edit at all, because it counts down
  toward a fixed date.

---

## 6. What the live data actually looks like

Measured against the production database (7,885 products), restricted to products that get a real
buyable product page (3,517 of them):

| | Count |
|---|---|
| `leadTime` flag on | 1,627 |
| Flag on **and** a real duration | 1,627 |
| **Flag on but no duration → "Available with lead time"** | **0** |
| Duration set but flag off | 1 |

**No live product page shows "Available with lead time" today.** Every product with the flag on also has a
number. The state is reachable in code, it is just not reached by current data.

Live durations in use, and what each prints:

| Days | Products | Shows as |
|---|---|---|
| 2 | 25 | 2 Working Days Lead Time |
| 3 | 48 | 3 Working Days Lead Time |
| 4 | 2 | 4 Working Days Lead Time |
| 5 | 211 | 1 Week Lead Time |
| 7 | 142 | 7 Working Days Lead Time |
| 10 | 383 | 2 Weeks Lead Time |
| 13 | 3 | 13 Working Days Lead Time |
| 15 | 131 | 3 Weeks Lead Time |
| 20 | 221 | 4 Weeks Lead Time |
| 25 | 39 | 5 Weeks Lead Time |
| 30 | 313 | 6 Weeks Lead Time |
| 35 | 68 | 7 Weeks Lead Time |
| 40 | 30 | 8 Weeks Lead Time |
| 45 | 6 | 9 Weeks Lead Time |
| 50 | 4 | 10 Weeks Lead Time |
| 60 | 2 | 12 Weeks Lead Time |

Roughly **46% of buyable products carry a lead time**. This is not a rare state — it is nearly half the
catalogue, and the "In Stock" green chip is the minority case on many category pages.

### Where "Available with lead time" *does* appear

**On product cards for POA products.** In the card component the code hard-codes it:

> if the product is POA → set the flag on and the duration to 0

Which lands exactly on the fallback. So every "Price on Application" product appearing in a Related
Products carousel or a category list shows a card reading **"Available with lead time"**. With 33 published
POA products, this is live and visible right now.

It is arguably the right message there — a POA product genuinely has no known delivery time — but it was
reached by accident, through a fallback meant for missing data.

---

## 7. Two quirks that will confuse you if nobody warns you

**The page loads showing the wrong thing.** The product page builds its working copy of the product only
*after* the page has mounted in the browser. Until then the lead-time calculation runs against nothing and
returns zero — so the first paint of every product page says **"In Stock"**, then corrects itself to the
lead time a moment later once the script runs.

You can see this by downloading the HTML for a lead-time product: `/badger-netting` has a 5-day lead time in
the database but its delivered HTML says "In Stock". This is a web-only artefact of how that page is built.
**The mobile app won't inherit it** — but do decide what the app shows while the product is still loading,
because "In Stock" is the worst possible default for a product that isn't.

**There is a fourth stock component that nobody uses.** `stock-message-header.js` implements the chip's
rule (flag *or* number) with the text line's styling. It is imported nowhere. If you find it while reading
the code, ignore it — it is dead.

---

## 8. Recommendations for the mobile app

1. **Use one component and one rule everywhere.** The web has three near-copies with two different rules and
   two different colours; that is the entire source of the confusion documented above. Pick the chip's rule
   (`flag on OR number > 0` means "there is a lead time") and apply it in every place.
2. **Decide what "Available with lead time" should say.** As data it means *"we don't know how long."* As
   copy it currently reads like a feature. Something like **"Made to order — contact us for timing"** says
   the true thing. No live product page depends on the current wording, so it is free to change.
3. **Make it live.** 727 products change stock state when an option is tapped. The indicator must re-render
   on every option change, and ideally be visible from where the shopper is tapping.
4. **Fix the pluralisation** (`6 Working Day` → `6 Working Days`) and consider dropping the weeks/days split
   entirely — "10 working days" is clearer than "2 Weeks" for a delivery estimate, and it removes the
   5 → 6 label jump.
5. **Size for 26 characters** in the chip, and decide whether it wraps or truncates on the narrowest phone.
6. **Pick one colour** for the lead-time state instead of grey-in-the-chip / orange-in-the-line, and use an
   icon that reads as *waiting* rather than a tick box.

---

## 9. Source references

- `src/utils/product/lead-time.js`
  - `getLeadTime` (line 34) — the three-step calculation in section 5
  - `getLeadTimeMessage` (line 92) — every string in section 3, including the "Available with lead time"
    fallback at line 98 and the pluralisation bug at line 106
  - `calculateWorkingDays` (line 7) — weekend-skipping day count
- `src/components/image-gallery/badges/badges.jsx` — the image chip; `leadTime || leadTimeDuration`
- `src/components/image-gallery/badges/badges.module.css` — chip colours and positions
- `src/components/stock-message/stock-message.js` — the text line; `parseInt(duration) > 0`
- `src/components/stock-message/stock-message.module.css` — green/orange line colours
- `src/components/product-card/icons/icons.js` — card label
- `src/components/product-card/product-card.js` (line 41) — the POA hard-code
- `src/templates/product/product.js` (line 365) — where the page computes the duration; state is set in
  `componentDidMount`, which is the first-paint quirk
- `src/components/stock-message-header/stock-message-header.js` — the unused fourth component
