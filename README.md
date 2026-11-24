# DetailBid Studio

A quote builder for auto detailing businesses — solo detailers and small
shops. Tagline: **"Stop undercharging. Quote every detail profitably."**

Packages, vehicle-size multipliers (plus per-size price overrides),
add-ons, and a condition adjustment, with the detailer's real costs,
profit, and true margin kept private — and a professional branded PDF
quote the customer can sign. The headline feature is **Profit Guard**:
set a target margin and the tool shows the minimum price that protects
it, warns plainly when a quote underpays you, and offers the exact fix
as a one-tap price adjustment. Quotes can also be **deposit-ready**
(your own payment link, deposit + remaining balance printed on the
quote), and a **follow-up queue** flags sent quotes that have waited
too long, with a ready-to-paste follow-up message for each. The paid
layer is professional documents and saved business data, never just
arithmetic: free detailing calculators already exist, so DetailBid Pro
sells the quote document, the price book, and the quote history.

Pricing: **$19/month founding-member pricing** for the first 25 customers,
locked in while the subscription stays active. Standard price is
**$29/month** after the founding spots are gone. The calculator and a
watermarked preview are free, and the first clean PDF quote is free;
Pro unlocks unlimited PDFs, the editable price book, saved quotes, the
quote dashboard, and backup export.

Plain static HTML/CSS/JS — no build step, no backend.

## Files

| File | What it is |
|---|---|
| `index.html` | Landing page: promise, how it works, free/Pro features, a static sample quote, founding pricing, privacy + billing/cancellation notes. |
| `app.html` | The tool: quote builder (customer/vehicle, packages × size multipliers or per-size overrides, add-ons, condition %/$/extra-hours, tax), Profit Guard (target margin, minimum price, one-tap adjustment), fuller cost model (travel, overhead, processing fees), deposit-ready quotes (your payment link), private numbers panel, watermarked preview, PDF quote (jsPDF via CDN), editable price book + override matrix, saved quotes + dashboard + follow-up queue, backup export/import + CSV export. |
| `README.md` | This file. |

## Before launch

- In `index.html`, replace the two `STRIPE_LINK` placeholder hrefs (hero +
  pricing) with the real Stripe subscription payment link for DetailBid
  Studio ($19/month founding price). In `app.html`, set the `STRIPE_LINK`
  constant at the top of the script — it feeds the Subscribe button in the
  Pro gate card.
- Contact email in the footer is `chrisla122787@gmail.com`.

## Free vs Pro

**Free (no payment, no account):**

- Full quote calculator — every input, live math, low-margin warning.
- **Profit Guard** — set your target margin; the private panel shows the
  minimum price that earns it, your earnings per working hour, and a
  plain warning with the exact shortfall when a quote underpays you.
  One tap adds the shortfall as a plain "Price adjustment" line on the
  quote (removable). The target, the warning, and every cost number
  stay private — the customer sees only the price.
- Watermarked quote preview rendered on screen, showing shop name,
  customer, vehicle, services, and prices.
- **One clean, unwatermarked branded PDF quote** (see "First quote free").

**Pro (subscription):**

- Unlimited clean branded PDF quotes: shop logo + contact info, itemized
  packages and add-ons, condition adjustment and tax lines when they
  apply, estimated time, 14-day validity, terms, and a customer
  acceptance block.
- **Deposit-ready quotes**: set a deposit (a % of the total or a fixed
  amount) and paste your own payment link (Stripe, Venmo, Cash App…).
  The PDF and preview print "Deposit to reserve your appointment: $X",
  your link, and "Remaining balance due at service: $Y". The customer
  pays you directly — DetailBid never handles the money. Saved quotes
  get a "Deposit received" toggle in the dashboard. No deposit
  configured, none of this appears.
- Editable price book: every package base price, vehicle-size multiplier,
  add-on price — plus an **override matrix** (an exact price per package
  per vehicle size that wins over base × multiplier; blank = use the
  formula) — saved in the browser and used by new quotes immediately.
  Saved quotes are **snapshots**: they keep the prices and totals they
  were saved with, so price-book edits never rewrite history.
- Saved quotes + quote dashboard (statuses Draft / Sent / Accepted /
  Declined / Completed; pipeline and accepted totals; load, duplicate,
  delete; deposit-received toggles).
- **Follow-up queue**: quotes in Sent status older than a configurable
  number of days (default 3) are listed under "Needs follow-up", each
  with a "Copy follow-up" button that builds a polite prefilled message
  (customer name, vehicle, quote total, shop name/phone) to paste into
  a text or email. Nothing is ever sent automatically.
- Backup export (JSON) / import-restore, and CSV export of quotes.

**The customer PDF and preview never show internal numbers.** The hourly
rate, product/supplies/travel/overhead costs, processing fees, profit,
margin, and target margin appear only in the private "Your numbers"
panel — the quote carries services and prices only. The one deliberate
exception: the deposit payment link is the detailer's own and is printed
on the quote by design, so the customer can pay the deposit.

## First quote free

Free users get exactly **one** complete branded PDF quote, unwatermarked:

- The counter lives in `localStorage` (`detailbid.freepdf.v1`) and is
  incremented **only after the PDF is successfully generated** — if the
  PDF library fails to load, the free quote is not consumed and the user
  is told so.
- After the free one, the PDF button asks for Pro. Pro users generate
  unlimited PDFs and the counter is never touched.

## ⚠️ MVP access note (read before launch)

Pro unlocking is **client-side only** for launch validation:

- The "I already subscribed — unlock Pro" button in `app.html` simply sets
  a `localStorage` flag (`detailbid.pro.v1 = "1"`) in the visitor's own
  browser. There is **no real payment verification** — no server checks
  the Stripe subscription, and anyone who finds the button can unlock Pro
  without paying.
- This is a deliberate MVP trade-off to validate demand cheaply on a $0
  static stack (the landing page says so in plain words: unlocking is on
  the honor system during opening validation).
- Before scaling past validation, replace this with a real entitlement
  check (e.g. a small serverless function that verifies the Stripe
  subscription by email before issuing an unlock token).

## The quote formula

All math is in `calculateQuote()` in `app.html`:

```
packagePrice  = override[package][size] if set, else round2(package base × vehicle-size multiplier)
subtotal      = packagePrice + Σ selected add-on prices
conditionAdj  = % mode: round2(subtotal × condition%)
              | fixed mode: conditionFixed $
              | hours mode: round2(extraHours × hourlyRate)
baseJobPrice  = subtotal + conditionAdj
jobPrice      = baseJobPrice + priceAdjustment   ← Profit Guard line, 0 unless applied
tax           = round2(jobPrice × tax%)
total         = jobPrice + tax
processingFee = round2(baseJobPrice × processing%)
estCost       = hourlyRate × (estHours + extraHours) + supplies + travel + overhead + processingFee
profit        = jobPrice − estCost               ← "Estimated job profit", not a guaranteed net
margin%       = profit ÷ jobPrice × 100          ← margin is a % of the PRICE
earnings/hour = profit ÷ (estHours + extraHours)
minPrice      = estCost ÷ (1 − targetMargin%)    ← Profit Guard minimum
shortfall     = max(0, minPrice − jobPrice)      ← what "Apply recommended price" adds
deposit       = % mode: round2(total × deposit%) | fixed mode: depositFixed $ (capped at total)
balance       = total − deposit
```

Margin is a share of the final price, not a markup on cost — the margin
shown is the share of the job price that is actually profit. Tax is
excluded from the profit/margin math: margin is measured on the job price
before tax, since tax is collected for the state, not earned. Processing
fees are charged on the pre-adjustment job price on purpose: applying the
Profit Guard recommendation then lands the margin exactly on target.
In "extra labor hours" condition mode, the extra hours raise both the
price (at the detailer's hourly rate) and the estimated cost hours — a
surcharge that only raised the price would overstate the profit.

Starter price book (all editable in Pro):

- Packages (sedan base): Express Wash $79 · Interior Detail $149 ·
  Exterior Detail $129 · Full Detail $249 · Paint Enhancement $449 ·
  Ceramic Coating Prep $349 · One-Step Paint Correction $399 · Two-Step
  Paint Correction $649 · Ceramic Coating $699 · Coating Maintenance
  $149 — each with a default estimated-hours value (1 / 3 / 2.5 / 5 / 8 /
  6 / 6 / 10 / 8 / 2) that fills in when the package is picked. The
  advanced packages (paint correction, ceramic coating) are editable
  placeholder defaults — every shop should set its own.
- Size multipliers: Coupe ×0.9 · Sedan ×1.0 · SUV ×1.25 · Truck ×1.25 ·
  3-Row SUV / Van ×1.4 — plus optional per-package, per-size price
  overrides in the price book matrix.
- Add-ons: Pet hair removal $45 · Engine bay cleaning $40 · Headlight
  restoration $60 · Odor treatment $55 · Clay bar treatment $50 · Seat
  extraction $65 · Excessive-soil surcharge $35.
- Condition presets: Light 0% · Moderate 10% · Heavy 20% (or a custom %),
  in % / fixed-$ / extra-hours modes.
- Profit Guard target margin: 40% by default (a setting, saved with the
  shop settings).

Worked example — the landing-page sample (2022 Ford Explorer, SUV):

- Full Detail: $249 × 1.25 = **$311.25**
- Add-ons: pet hair $45 + engine bay $40 = $85 → subtotal **$396.25**
- Condition: Light (0%) → job price **$396.25**; tax 0% → total **$396.25**
- Private view: 5 hrs × $35/hr + $18 products = $193 cost → profit
  **$203.25**, margin **51.3%**

Warnings: margin under 10% (on a priced job) shows a plain-language
caution in the private panel.

## Data & privacy

Everything lives only in the visitor's own browser (`localStorage`):

| Key | Contents |
|---|---|
| `detailbid.settings.v1` | Shop name/phone/email + logo (data URL), target margin %, deposit type/%/fixed amount, payment link, follow-up days. |
| `detailbid.pricebook.v1` | Packages, multipliers, add-ons, and per-size price overrides (user's edited prices). Price books saved before the advanced packages existed are merged forward on load: missing packages are added, existing prices are kept. |
| `detailbid.quotes.v1` | Saved quotes (input + computed snapshots, name, status, date, `sentAt` when marked Sent, `depositReceived` flag). |
| `detailbid.pro.v1` | Pro unlock flag (see MVP note above). |
| `detailbid.freepdf.v1` | Free-PDF counter (0 or 1). |

Nothing is sent to a server. The app tells users plainly: "Your data is
stored only in this browser on this device — export a backup regularly."
Backup export downloads a JSON file with settings + price book + quotes;
import validates the file (it must be a DetailBid Studio backup with a
quotes list and, if present, a well-formed price book) **before** changing
anything, and rejects malformed files with state untouched. CSV export
lists quotes with their saved totals.

Quote statuses: Draft, Sent, Accepted, Declined, Completed. Dashboard
totals: active pipeline = Draft + Sent quote totals; accepted work =
Accepted + Completed quote totals. Marking a quote Sent stamps its
`sentAt` date, which is what the follow-up queue counts from (falling
back to the saved date for older quotes).

## Deploy

Any static host works — copy the three files (or at least the two HTML
files) to the host as-is:

- **Netlify / Vercel:** drag-and-drop the folder, or connect the repo; no
  build command, publish directory = this folder.
- **Cloudflare Pages:** same — framework preset "None", no build command.
- **GitHub Pages:** fine for demos/preview only — GitHub Pages terms
  prohibit primarily commercial SaaS use, so don't run the paid product
  from there.

The only external request the app makes is the jsPDF CDN script in
`app.html` (cdnjs). If it fails to load, everything except the PDF button
still works and the user gets a clear message — and a free user's free
PDF is not consumed.
