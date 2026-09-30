# The Four-Month Model

A client-facing, lead-generating ROI calculator built for **KMS Pharma** (contact: Dr Ganesan). It answers one question for a formulation, regulatory, or BD lead at a generics company: *what is an in-house BA/BE study plus dossier development, under one roof, actually worth on this specific molecule?*

It is a single static HTML file. No framework, no build step, no server-side code. Everything, markup, styling, state machine, financial model, and PDF export, lives in [`index.html`](index.html) (~1,450 lines) and is deployed as-is to Vercel.

---

## Table of contents

- [What this page is for](#what-this-page-is-for)
- [Quick start](#quick-start)
- [Project structure](#project-structure)
- [The user flow (3-step wizard)](#the-user-flow-3-step-wizard)
- [The financial model](#the-financial-model)
  - [Inputs](#inputs)
  - [V1: earlier revenue capture](#v1-earlier-revenue-capture)
  - [V2: entrant-rank shift (with margin erosion)](#v2-entrant-rank-shift-with-margin-erosion)
  - [V3: cost & risk avoidance](#v3-cost--risk-avoidance)
  - [Gross value, risk adjustment, and ROI multiple](#gross-value-risk-adjustment-and-roi-multiple)
  - [NPV, time-adjusted](#npv-time-adjusted)
  - [Worked example](#worked-example)
- [The timeline bridge (before/after animation)](#the-timeline-bridge-beforeafter-animation)
- [Markets, currencies, and dosage forms](#markets-currencies-and-dosage-forms)
- [Lead-gen mechanics and persistence](#lead-gen-mechanics-and-persistence)
- [PDF report generation](#pdf-report-generation)
- [Call-to-action links](#call-to-action-links)
- [Design system](#design-system)
- [Accessibility and mobile](#accessibility-and-mobile)
- [Deployment](#deployment)
- [Customizing this for another client or molecule](#customizing-this-for-another-client-or-molecule)
- [Known limitations / what still needs doing before this is fully "live"](#known-limitations--what-still-needs-doing-before-this-is-fully-live)
- [Editing conventions for this file](#editing-conventions-for-this-file)
- [Credits](#credits)

---

## What this page is for

This is a **lead magnet**, not an internal tool. The audience is a formulation, regulatory, or business-development lead at a generics manufacturer who is deciding whether to run BA/BE bioequivalence studies and dossier development through an external CRO, or in-house with KMS Pharma.

The page makes one argument, in three layers:

1. **Hook (hero):** filing sooner, not "eventually," is where the money is. A late filer doesn't just miss a few months of sales, they permanently land in a worse position on the price/market-share curve for the rest of the product's life.
2. **Credibility bridge:** a concrete, step-by-step breakdown of *why* an in-house facility is faster than an outsourced CRO (RFP/contract, clinic queue, method development, protocol/EC, CSR turnaround), shown as an animated before/after comparison rather than a wall of numbers.
3. **Calculator:** the visitor plugs in their own molecule's numbers (market size, expected filing position, margin, etc.) and gets a personalized dollar figure for what acceleration is worth to them, gated behind an email capture, with a "Book a call" CTA as the natural next step and a structured PDF they can take to their own stakeholders.

## Quick start

There is no build step. To work on this locally:

```bash
git clone git@github.com:aitoolsteejay/kms-pharma-roi.git
cd kms-pharma-roi
python3 -m http.server 8080
```

Then open `http://localhost:8080/index.html`.

> **Why not just double-click the file?** `localStorage` (used for the "unlock once, stay unlocked" behavior and returning-visitor state restore) is disabled for `file://` URLs in most browsers. Serve it over `http://` (any static server works: `python3 -m http.server`, `npx serve`, VS Code's Live Server, etc.) to test that behavior. Everything else works fine straight from disk.

No `npm install`, no dependencies to fetch locally. The only external requests the page makes are:
- Google Fonts (`Fraunces`, `IBM Plex Sans`, `IBM Plex Mono`)
- jsPDF and jspdf-autotable, loaded from cdnjs (only used when the visitor clicks "Download PDF report")

## Project structure

```
kms-roi-calculator/
├── index.html                    # Everything: markup, CSS, and JS. The only file that matters.
├── assets/
│   ├── kms-logo-cropped.jpeg     # Used in the topbar <img> and drawn into the PDF header
│   └── kms-logo.jpeg             # Uncropped source logo, kept for reference/future re-crops
├── vercel.json                   # Cache headers + clean URLs for the Vercel static deploy
├── .gitignore                    # .DS_Store, .vercel, node_modules
└── README.md                     # This file
```

There is deliberately no `package.json`, no bundler config, and no CSS/JS split. The brief called for a single deployable artifact a non-engineer can hand to Vercel and forget about. If you're tempted to split this into components, read [Editing conventions](#editing-conventions-for-this-file) first.

## The user flow (3-step wizard)

The calculator is a single `<div class="calc-grid">` whose visible content is controlled entirely by CSS attribute selectors keyed on `data-step`:

```css
.calc-grid[data-step="1"] { /* inputs only: hides .lead-gate and .output */ }
.calc-grid[data-step="2"] { /* email gate only: hides .panel and .output */ }
.calc-grid[data-step="3"] { /* results only: hides .lead-gate and #seeResultsBtn */ }
```

There is no client-side router and no page reload between steps, `goToStep(n)` just writes `n` to the grid's `data-step` attribute and, when moving to step 3, calls `recalc()` to compute and paint the results.

1. **Step 1, Inputs.** The visitor configures their molecule: market, dosage form, market value, expected entrant position, gross margin, and (behind an "Advanced assumptions" `<details>`) months of acceleration, rank-shift, remaining lifecycle, margin erosion, approval probability, repeat-study probability, discount rate, CRO/in-house costs, programme cost, baseline filing date, and the full editable price/share-by-rank curve. A live demo scenario (US, oral solid, $80M market) is preloaded via `loadDemo()` so the page never looks empty.
2. **Step 2, Email gate.** Clicking "See what this is worth →" advances to step 2, a single-field form (`#gateForm` / `#gateEmail`). Submitting it calls `setUnlocked()` (writes `kms_roi_unlocked=1` to `localStorage`) and advances to step 3. There's a "back" link to return to step 1 without losing the entered values.
3. **Step 3, Results.** `recalc()` renders the headline number, the V1/V2/V3 stacked bar, the full value breakdown table with inline "i" tooltips, the "Book a call" CTA, and the "Download PDF report" button.

Once a visitor has unlocked the page once, `kms_roi_unlocked` and a full snapshot of their inputs (`kms_roi_state`, see [Lead-gen mechanics](#lead-gen-mechanics-and-persistence)) persist in `localStorage`. On a later visit, the page skips straight to step 3 with their exact scenario restored, it does not silently reset them to the generic US demo.

## The financial model

All financial math lives in one function, `computeScenario()`, which both the live page (`recalc()`) and the PDF generator (`generatePDF()`) call. This is intentional: there is exactly one source of truth for the numbers, so the on-screen figures and the PDF can never drift apart.

### Inputs

| Input | Element id | Notes |
|---|---|---|
| Target market | `market` | 13 options, see [Markets](#markets-currencies-and-dosage-forms) |
| Dosage form | `dosage` | 5 options, drives repeat-study probability and cost defaults |
| Reporting currency | `currency` | 12 ISO currencies; defaults to the market's native currency but is independently editable |
| Annual market value of the molecule | `marketValue` / `marketValueRange` | Slider $5M–$500M, or type an exact figure |
| Expected entrant position without acceleration | `entrantRank` | 2nd through "7th or later" (capped at rank 6 for curve lookups) |
| Gross margin, year 1 | `grossMarginRange` | 10%–80% slider. This is the *starting* margin; see V2 below for how it erodes |
| Months of acceleration | `monthsAccel` | Default 4 (P50); 6–7 reflects the "repeat-study protection" scenario |
| Entrant-rank positions gained | `rankShift` | 0, 1, or 2 positions moved up the filing queue |
| Remaining product lifecycle (years) | `lifecycle` | 1–15 years |
| Gross margin erosion, per year | `marginErosion` | 0%–10%, default 4%. Compounds every year across the lifecycle |
| P(approval, first cycle) | `approvalProb` | Default 80% |
| P(repeat study) | `repeatProb` | Auto-set by dosage form, manually overridable |
| Discount rate, r | `discountRate` | Default 13%, used for the NPV calculation |
| CRO study cost | `croCost` | Auto-set by dosage form |
| KMS in-house study cost | `inHouseCost` | Auto-set by dosage form |
| Incremental KMS programme cost | `programmeCost` | Denominator for the ROI multiple; defaults to in-house BA/BE + dossier cost |
| Baseline filing target | `baselineFiling` | Month/year picker; defaults to 18 months out |
| Price & share by entrant rank | editable table | 6 rows (rank 1–6), each with a price-as-%-of-brand and a market-share-%; auto-populated per market, fully editable |

Any field marked "auto-set by dosage form" tracks a `data-auto` flag (`'true'`/`'false'`) on its `<input>`, once a visitor manually edits it, switching dosage form again won't silently overwrite their number.

### V1: earlier revenue capture

```
V1 = marketValue × revShareNew × grossMargin × (monthsAccel / 12)
```

Extra profit from selling sooner: market value times the price-and-share you get *at the new (accelerated) rank*, times margin, times the fraction of a year those extra months represent. This value is realized in year one, so it uses the year-1 gross margin with no erosion applied.

### V2: entrant-rank shift (with margin erosion)

This is usually the largest of the three values, and the one that most needs explaining: it is not "a few months of extra sales." It is the value of *permanently* holding a better price and market-share position for the rest of the product's commercial life, because you filed while there were fewer competitors ahead of you in the queue.

Naively, you might compute this as a single flat number:

```
incrementalRevenue = marketValue × revShareNew − marketValue × revShareOld
V2_flat = incrementalRevenue × grossMargin × lifecycleYears        (NOT what the code does)
```

But that assumes margin holds constant for the entire remaining lifecycle, which is unrealistic: as more competitors enter and prices erode, margin compresses year over year. So `computeScenario()` instead compounds a declining margin, year by year, across the lifecycle:

```js
for (var t = 1; t <= lifecycleYears; t++) {
  var marginYear = grossMargin * Math.pow(1 - marginErosion, t - 1);
  V2 += incrementalRevenue * marginYear;
}
```

At the default 4%/year erosion rate, a 45% year-1 margin has fallen to roughly 38% by year 5. The live results page surfaces this explicitly under the V2 line (e.g. *"Margin 45% → 38% by year 5, eroding 4%/year."*) rather than burying it in the math. Setting `marginErosion` to 0% reproduces the flat-margin calculation exactly, so this is a strict generalization of the simpler model, not a behavior change when erosion is turned off.

### V3: cost & risk avoidance

```
delayValue3mo = marketValue × revShareOld × grossMargin / 12 × 3
V3 = (croCost − inHouseCost) + repeatProb × (croCost + delayValue3mo)
```

Two components:
1. **Direct cost saving**: what you pay an outside CRO minus what the in-house study costs.
2. **Repeat-study protection**: the probability a bioequivalence study needs to be repeated (dosage-form-dependent, e.g. 28% for topicals vs. 12% for oral solids), times the cost of repeating it *plus* the value of the roughly 3-month filing delay a repeat typically causes if run externally. In-house, a repeat is a re-slot; externally, it means re-queueing behind the CRO's other clients.

### Gross value, risk adjustment, and ROI multiple

```
gross         = V1 + V2 + V3
riskAdjusted  = gross × approvalProb
roiMultiple   = riskAdjusted / programmeCost
```

`riskAdjusted` is the headline number shown at the top of the results page. `roiMultiple` is riskAdjusted expressed as a multiple of what the acceleration programme itself costs.

### NPV, time-adjusted

A more conservative, discounted view of the same cash flows:

```js
function disc(t) { return 1 / Math.pow(1 + r, t); }   // r = max(discountRate, -0.99), guards against -100%+ inputs

npv = V1 * disc(0.5) + V3 * disc(0.5);                // V1/V3 land mid-year-one
for (t = 1; t <= lifecycleYears; t++) {
  npv += v2ByYear[t - 1] * disc(t);                    // each year's *already-eroded* V2 slice, discounted at year t
}
npvRiskAdj = npv * approvalProb;
```

Critically, the NPV loop discounts each year's actual (eroded) V2 contribution, not an average, so the margin-erosion assumption flows through consistently into the conservative NPV figure as well as the headline gross number.

### Worked example

With the preloaded demo scenario (US market, oral solid, $80M annual market value, 6th-to-file without acceleration, 45% year-1 margin, 4 months acceleration, 2 rank positions gained, 5-year remaining lifecycle, 4%/year margin erosion, 80% approval probability, 13% discount rate):

- V2 alone is roughly **$4.1M** (vs. ~$4.4M if margin were held flat at 45% for all 5 years, illustrating the erosion effect)
- Headline risk-adjusted value comes out to roughly **$3.8M**
- NPV, time-adjusted, comes out to roughly **$2.8M**

(Exact figures depend on the editable price/share curve and will shift slightly as the model is tuned; use the live demo, not this README, as the source of truth.)

## The timeline bridge (before/after animation)

The credibility section under the hero answers "why is in-house 4 months faster?" with five concrete steps, defined once in JS and reused for both the summary numbers and the (collapsed-by-default) detail table:

```js
var CRO_STEPS = [
  { label: 'RFP / contract',     wk: 4.5 },
  { label: 'Clinic queue',       wk: 6   },
  { label: 'Method dev+val',     wk: 5.5 },
  { label: 'Protocol/EC',        wk: 3   },
  { label: 'CSR turnaround',     wk: 5   },
];
CRO_TOTAL = 24 weeks   // sum of the above
SAVE_TOTAL = 18 weeks  // KMS's claimed saving
KMS_TOTAL  = 6 weeks   // CRO_TOTAL - SAVE_TOTAL: in-house equivalent timeline
```

The visible UI is a single animated before/after bar (`runBeforeAfterAnim()`, an `IntersectionObserver`-triggered width transition plus a count-down number), because earlier iterations of this section tried to gamify all five steps individually and testing showed that was "too much information." The full 5-row breakdown is still there for a visitor who wants it, behind a native `<details class="bridge-table">` element that animates its rows in via `runRowCascade()` the first time it's expanded.

All animations respect `prefers-reduced-motion` (checked both in CSS and in JS via `matchMedia`) and are skipped/instant for visitors who've requested reduced motion.

## Markets, currencies, and dosage forms

**13 markets**, each with a 6-point price-erosion curve (`P`, price as % of the original brand, by filing rank 1–6) and an expected review timeline:

| Code | Market | Review months |
|---|---|---|
| US | United States (ANDA) | 10 |
| EU | European Union (DCP/MRP) | 14 |
| UK | United Kingdom (MHRA) | 12 |
| CA | Canada (Health Canada) | 12 |
| AU | Australia (TGA/PBS) | 12 |
| JP | Japan (PMDA) | 12 |
| BR | Brazil (ANVISA) | 14 |
| IN | India (CDSCO) | 9 |
| ZA | South Africa (SAHPRA) | 13 |
| SA | Saudi Arabia / GCC (SFDA) | 10 |
| CN | China (NMPA) | 16 |
| MX | Mexico (COFEPRIS) | 12 |
| RoW | Rest of World | 15 |

Market share by rank (`S_DEFAULT = [40, 25, 19, 15, 11, 9]`, percent of the addressable generic market) is the same starting point across all markets, but is fully editable per scenario in the "Price & share by entrant rank" table.

**12 currencies** (`CURRENCY_ORDER`): USD, EUR, GBP, CAD, AUD, JPY, BRL, INR, ZAR, SAR, CNY, MXN. The live page uses native Unicode symbols (₹, ¥, €, £, …); the PDF uses ASCII-safe ISO codes instead (`INR 2.7M` rather than `₹2.7M`), see [PDF report generation](#pdf-report-generation) for why.

**5 dosage forms**, each with a repeat-study probability and cost defaults:

| Dosage form | P(repeat study) | CRO cost | In-house cost |
|---|---|---|---|
| Oral solid (IR) | 12% | $320,000 | $220,000 |
| Oral liquid | 15% | $300,000 | $200,000 |
| Topical / semisolid | 28% | $380,000 | $260,000 |
| Injectable (complex generic) | 18% | $450,000 | $320,000 |
| Modified release | 25% | $360,000 | $250,000 |

> Dollar figures throughout (the market-value slider, and the CRO/in-house cost defaults) are calibrated for USD-scale amounts. If you switch currency, adjust these manually, the page does not currently auto-convert.

## Lead-gen mechanics and persistence

Two `localStorage` keys drive returning-visitor behavior, both wrapped in `try/catch` (private browsing / storage-blocked contexts degrade gracefully to "always show the demo, never persist"):

- **`kms_roi_unlocked`** (`'1'` or absent): set the moment the email-gate form is submitted. Nothing is validated or transmitted, it's purely a client-side flag.
- **`kms_roi_state`**: a JSON snapshot of every input (market, dosage, currency, all advanced assumptions, the editable price/share curve, and which cost fields were auto vs. manually set), written by `saveState()` on every `recalc()` while unlocked, and restored by `restoreState()` on page load.

On load, the init sequence is:

```js
if (isUnlocked() && restoreState()) {
  // returning, already-unlocked visitor: restore their exact scenario, skip to step 3
} else {
  // first-time visitor, or corrupted/missing state: load the generic demo
  loadDemo();
  if (isUnlocked()) goToStep(3);   // unlocked but no valid state → demo at step 3, not a crash
}
```

`restoreState()` is defensive: missing fields fall back to sane defaults (e.g. a state saved before the margin-erosion feature existed defaults `marginErosion` to 4%), and any JSON parse failure or missing `market` key causes it to return `false` and fall through to the demo rather than throwing.

## PDF report generation

Clicking "Download PDF report" builds a client-side PDF via **jsPDF 4.2.1** + **jspdf-autotable 5.0.8** (pinned CDN versions, loaded from cdnjs). The report is deliberately structured, not a screenshot:

1. **Header**: KMS logo (drawn to an off-screen `<canvas>` and embedded as a JPEG), report title, generation date.
2. **Headline result box**: the risk-adjusted value and ROI multiple, large and first.
3. **Section 1, Inputs Used**: every input from the table above, including the year-1/final-year gross margin and the erosion rate.
4. **Section 2, Timeline Bridge**: the 5-step CRO breakdown, the in-house equivalent, and the caveat about repeat-study delay.
5. **Section 3, Price & Share Curve**: the full 6-rank table as configured for this scenario.
6. **Section 4, Value Breakdown**: V1/V2/V3 with their formulas spelled out in words, gross, risk-adjusted, and NPV.
7. **Section 5, Facility Credential / disclaimer**: placeholder credential line and the illustrative-model legal disclaimer (see [Known limitations](#known-limitations--what-still-needs-doing-before-this-is-fully-live)).

Manual pagination is handled by two small helpers, `ensureSpace(neededHeight)` (adds a page if the next block won't fit) and `heading(num, title)` (numbered section headers), since jsPDF has no automatic flow layout.

**Why the PDF uses `INR` instead of `₹`:** jsPDF's built-in standard fonts only support WinAnsi encoding (Latin-1 plus a handful of extras). Characters outside that range, ₹ (U+20B9) is the one that bit us, render as corrupted glyphs (confirmed by decoding the raw PDF bytes). The live page still uses native Unicode symbols since browsers render those fine; only the PDF path substitutes ISO currency codes (`pdfSym = s.currency + ' '`).

## Call-to-action links

The real KMS Pharma Microsoft Bookings link is wired into three places, each pointing at:

```
https://bookings.cloud.microsoft/bookwithme/user/06b9a3e9ce3c46a4a5c731998ded5e0f@kmshc.com/meetingtype/zqRCFnOFHUaX7y0k6FWNaA2?anonymous&ismsaljsauthenabled&ep=mlink
```

1. **Topbar** (`.topbar-cta`): persistent "Book a call →" button, visible on all three wizard steps.
2. **Hero** (`.hero-cta-link`): a subtle "Prefer to talk it through? Book a call →" link for visitors who don't need the calculator to decide.
3. **Results page**: the primary CTA ("Book a call to discuss your molecule →"), placed above the now secondary-styled "Download PDF report" button, since booking a call is the stronger conversion action once someone has seen their number.

All three open in a new tab (`target="_blank" rel="noopener"`). If this link ever needs to change, it currently exists as a literal string in three places in `index.html`, search for `bookings.cloud.microsoft`.

## Design system

Everything is theme-tokenized via CSS custom properties on `:root` (see the top of the `<style>` block): `--bg`, `--surface`, `--surface-2`, `--text`, `--text-soft`, `--text-faint`, `--border`, `--border-soft`, `--accent` (teal), `--gold` (amber, used for emphasis/highlight numbers), `--danger`, `--focus`. There is **no dark mode**, this was an explicit, deliberate choice: the page must always render on a white background regardless of the visitor's OS theme, so all `prefers-color-scheme: dark` CSS was removed rather than maintained.

Typography: `Fraunces` (serif, headings), `IBM Plex Sans` (body), `IBM Plex Mono` (all numeric figures, via the `.num` class, for tabular alignment).

## Accessibility and mobile

- All interactive controls meet the 44px minimum touch-target guideline. Where a visual element is smaller than that (e.g. the 15px "i" info-tooltip buttons), an invisible `::before` pseudo-element expands the hit area to 36px+ without changing how it looks.
- Info tooltips (`.info-tip`) are keyboard-accessible (`:focus-visible`) and dismissible via outside-click or <kbd>Escape</kbd>, not just hover.
- All scroll-triggered and count-up/count-down animations check `prefers-reduced-motion` and skip straight to the end state when it's set.
- A full mobile-optimization pass has been done on layout, spacing, and tap targets across all three wizard steps.

## Deployment

This repo deploys to Vercel as a static site, no framework preset, no build command. [`vercel.json`](vercel.json) sets:

- `cleanUrls: true` and `trailingSlash: false`
- Long-lived immutable caching (`max-age=31536000`) for everything under `/assets/`
- No-cache, must-revalidate for `/index.html` itself, so content edits go live immediately without a manual cache purge

To deploy: connect this GitHub repo (`aitoolsteejay/kms-pharma-roi`) to a Vercel project with the framework preset set to "Other" and no build command, Vercel will serve `index.html` and `assets/` as-is.

## Customizing this for another client or molecule

Everything is a constant near the top of the inline `<script>` block, in this order: `MARKETS`, `MARKET_ORDER`, `S_DEFAULT`, `CURRENCIES`, `CURRENCY_ORDER`, `DOSAGE`, `CRO_STEPS` / `CRO_TOTAL` / `SAVE_TOTAL` / `KMS_TOTAL`. Common changes:

- **Add a market**: add an entry to `MARKETS` (label, 6-point `P` curve, `reviewMonths`, ISO `currency`) and its key to `MARKET_ORDER`.
- **Add a currency**: add an entry to `CURRENCIES` (`symbol`, `label`) and its key to `CURRENCY_ORDER`.
- **Change the CRO timeline steps or savings claim**: edit `CRO_STEPS` and `SAVE_TOTAL`.
- **Change branding**: replace `assets/kms-logo-cropped.jpeg`, the `<title>`/`<meta description>` in the `<head>`, the hero hook copy, and the footer credential line (currently a placeholder, see below).
- **Change the booking link**: search for `bookings.cloud.microsoft` (3 occurrences) and replace.
- **Change default assumptions**: `loadDemo()` and `resetAdvancedOnly()` both set the same default values; keep them in sync if you change one.

## Known limitations / what still needs doing before this is fully "live"

- **The email-gate form does not go anywhere.** `#gateForm`'s submit handler sets the unlock flag and saves scenario state locally; it does not transmit the email address to a CRM, webhook, or spreadsheet anywhere. Before this is used for real lead capture, wire `gateForm`'s submit handler to an actual endpoint (a serverless function, a form service, a CRM webhook, etc.).
- **The footer credential line is a placeholder.** It literally says *"Replace this credential line and the PDF's facility details with KMS's own before this goes live."* Do that before sharing this externally.
- **No currency auto-conversion.** Dollar-denominated defaults (market value slider range, CRO/in-house cost defaults) are calibrated for USD-scale figures and must be adjusted manually after switching currency.
- **No automated test suite.** This is a single static file with no build step; verification is done by serving it locally and testing the flow in a browser (see [Quick start](#quick-start)).

## Editing conventions for this file

This project intentionally stays a single HTML file with no build step, so a non-engineer can redeploy it by editing one file and pushing. If you're adding to it:

- Keep `computeScenario()` as the **only** place financial math happens. Both `recalc()` (live page) and `generatePDF()` must read from its return value, never duplicate a formula.
- No em dashes anywhere in the file (a hard style constraint for this project); en dashes in number ranges (e.g. "10-14 weeks") are fine.
- Every commit that touches financial logic should be verified live in a browser (values change as expected, no `NaN`/`Infinity`, PDF still generates) before it's pushed, this file has a history of subtle bugs (dosage-form resets clobbering manual edits, extreme discount rates producing `NaN`, returning visitors losing their configured scenario) that were only caught by re-testing the full flow end to end.

## Credits

Built for **KMS Pharma** (client contact: Dr Ganesan) by Sanyam (sanyam@myntmore.com), with Claude Code.
