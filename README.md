# Schbang Tee — Claim Yours

Internal merch drop page for Schbang employees to browse the 2026 limited-edition tee and claim their preferred fit and size. Built as a single static HTML page backed by a Google Apps Script API and Google Sheets database.

---

## How It Works

### Flow

```
Employee Code → Name Lookup → Email OTP Verification → Fit & Size Selection → Confirmation Email
```

1. **Employee Lookup** — User enters their Schbang employee code (e.g. `SDSPL001`). The page hits the Apps Script API with `?action=lookup&empCode=...` and returns the employee's name. If they've already submitted, it shows their current selection and remaining change count (max 5).

2. **Email Verification** — User enters their `@schbang.com` email. The API (`?action=sendOtp`) generates a 6-digit OTP and emails it. The frontend enforces the `@schbang.com` domain.

3. **OTP Entry** — 6-digit input with auto-advance, paste support, and auto-submit on completion. 30-second resend cooldown. Verified via `?action=verifyOtp`.

4. **Fit & Size Selection** — User picks Regular or Oversized, then a size. Oversized shows a "size down" recommendation. Submitted via `?action=submit`. A confirmation email is triggered via `?action=sendConfirmation`.

5. **Success** — Shows the saved selection and notifies the user they can return and change up to 5 times.

### Re-submission Logic

Employees can revisit and change their selection up to **5 times**. The lookup step detects prior submissions and shows the current choice with remaining changes. On the 5th change, the form locks.

### Deadline

A countdown timer targets `2026-04-30T23:59:59+05:30`. Once past, the claim form is replaced with a "Submissions Closed" message. The countdown runs client-side with `setInterval`.

---

## Architecture

```
index.html          ← Entire frontend (HTML + CSS + JS, no build step)
images/
  common/           ← Shared product shots (collar, front print, back print)
  regular-male/     ← Regular fit, male model
  regular-female/   ← Regular fit, female model
  oversized-male/   ← Oversized fit, male model
  oversized-female/ ← Oversized fit, female model
  logo.png          ← Navbar logo
```

**No frameworks, no bundler, no dependencies.** The only external resources are Google Fonts (Syne, DM Sans, DM Mono, Gotu) and the Vimeo Player API for the background video.

### API

All backend logic lives in a single Google Apps Script web app deployed as `exec`. The frontend communicates via GET requests with query params:

| Action | Params | Returns |
|---|---|---|
| `lookup` | `empCode` | `{ success, name, empCode, alreadySubmitted?, existingFit?, existingSize?, changeCount? }` |
| `sendOtp` | `email, empCode` | `{ success }` |
| `verifyOtp` | `email, otp` | `{ success }` |
| `submit` | `empCode, size, fit, email` | `{ success, name, fit, size }` |
| `sendConfirmation` | `email, name, empCode, fit, size` | `{ success }` |

All calls go through the `apiCall()` helper which builds the URL with `URLSearchParams` and follows redirects (required for Apps Script).

---

## Gallery & Size Chart

### Gallery

Images are defined in the `IMAGES` object, keyed by `common`, `regular.male`, `regular.female`, `oversized.male`, `oversized.female`. Two toggle groups control `currentFit` and `currentGender`. Switching the fit in the gallery also syncs the size chart toggle.

Clicking any image opens a full-screen lightbox (closes on click or `Escape`).

### Size Chart

Two datasets in `SIZE_DATA` — `regular` (XS–XXL) and `oversized` (XS–3XL). The table re-renders on toggle. The oversized chart includes a 3XL row.

---

## Styling

- **Grid**: 12-column, 80px margins (40px tablet, 20px mobile)
- **Type scale**: Syne (display/headings), DM Sans (body), DM Mono (data/inputs), Gotu (reserved for Hindi if needed)
- **Colors**: Monochrome — pure black background, white text, gray scale for hierarchy
- **Responsive**: Three breakpoints at 1024px, 768px, and 400px. Gallery goes 3 → 2 → 1 column. OTP inputs and timer blocks scale down.
- **Animations**: Scroll-triggered reveal (IntersectionObserver), form step transitions (`fadeSlideUp`), countdown blink, scroll indicator float

---

## Setup

1. **Clone the repo** and drop your product images into the `images/` folders matching the structure above.

2. **Deploy the Google Apps Script** — create a script bound to your Google Sheet with the 5 actions listed above. Deploy as a web app with "Anyone" access. Copy the deployment URL.

3. **Paste the URL** into the `API_URL` constant at the top of the `<script>` block:
   ```js
   const API_URL = 'https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec';
   ```

4. **Update the deadline** if needed:
   ```js
   const DEADLINE = new Date('2026-04-30T23:59:59+05:30');
   ```

5. **Update the Vimeo embed** — replace the iframe `src` with your video URL if different.

6. **Host anywhere** — drop on Netlify, Vercel, GitHub Pages, or any static host. No server required beyond the Apps Script backend.

---

## Google Sheet Schema (Expected)

The Apps Script expects a sheet with at least these columns:

| Employee Code | Name | Email | Fit | Size | Change Count | Timestamp |
|---|---|---|---|---|---|---|
| SDSPL001 | Jane Doe | jane@schbang.com | Regular | M | 1 | 2026-04-01T... |

OTPs are stored temporarily (likely in a separate sheet or Script Properties) and expire after verification.

---

## Notes

- All API calls are GET-based because Apps Script `doGet()` is simpler to deploy with public access. Sensitive operations are gated behind OTP verification.
- The confirmation email is fire-and-forget — a failed send is caught silently so it doesn't block the success state.
- The page is fully functional with JS disabled up to the gallery/chart (static content), but the form requires JS.
- Vimeo embed uses `background=1&autoplay=1&muted=1&loop=1` for a seamless hero video. On mobile, the iframe is scaled 160% and centered to crop letterboxing.
