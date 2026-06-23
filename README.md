# ExposureBoost — Shop + Order Form + Stripe + Google Sheets

## How the pieces fit

```
Shop (GitHub Pages)          Order form + API (Cloudflare Worker)        Google Sheet
index.html, success.html  →  checkout.exposureboost.co.uk           →  your spreadsheet
  cart in localStorage          GET  /         order form (name/email/phone/logo)
  "Checkout" redirects with     POST /create   logo→R2, create Stripe Checkout
  ?cart=metal:1,stand:2         GET  /logo/:k  serves logo for the sheet
                                POST /webhook  Stripe→ appends paid order row
```

- **Nothing secret is in GitHub.** The shop only contains the branded link
  `https://checkout.exposureboost.co.uk`. Stripe keys, the Sheets URL and the
  uploaded logos all live on Cloudflare.
- **Prices are enforced in the Worker** (`PRODUCTS`), so the cart can't be edited
  to change what someone pays.
- **Orders are saved only after successful payment** (via the Stripe webhook):
  name, email, phone, products, total, and the **logo** (stored in R2, shown in
  the sheet with `=IMAGE()`).

---

## Setup — do these in order

### 1. Google Sheet (Apps Script)
1. Open your sheet → **Extensions → Apps Script**.
2. Paste `sheets/Code.gs`. Set `SHEET_TOKEN` to a random password.
3. **Deploy → New deployment → Web app**: execute as **Me**, access **Anyone**.
4. Copy the **web-app URL** (looks like `https://script.google.com/macros/s/AKfy.../exec`).

### 2. Cloudflare Worker
From the `worker/` folder:
```bash
npx wrangler r2 bucket create exposureboost-logos
npx wrangler deploy
npx wrangler secret put STRIPE_SECRET_KEY        # sk_live_... (or sk_test_... to test)
npx wrangler secret put SHEET_WEBHOOK_URL         # the Apps Script URL from step 1
npx wrangler secret put SHEET_TOKEN               # same password as in Code.gs
```
Then in `wrangler.toml` set `SITE_URL` to your GitHub Pages shop URL and `npx wrangler deploy` again.

> Prefer the dashboard? **Workers & Pages → your worker → Settings → Variables**
> add the same secrets there, and **Settings → Bindings** add the R2 bucket as `LOGOS`.

**Custom domain:** Worker **Settings → Domains & Routes → Add → Custom domain**
→ `checkout.exposureboost.co.uk`. (Your domain must be on Cloudflare.)

### 3. Stripe webhook (so paid orders reach the sheet)
1. Stripe Dashboard → **Developers → Webhooks → Add endpoint**.
2. URL: `https://checkout.exposureboost.co.uk/webhook`
3. Event: **`checkout.session.completed`**.
4. Copy the **Signing secret** (`whsec_...`) and add it to the Worker:
   ```bash
   npx wrangler secret put STRIPE_WEBHOOK_SECRET
   ```

### 4. Shop (GitHub Pages)
- `index.html` already points at `https://checkout.exposureboost.co.uk`
  (`ORDER_FORM_URL`). If your order-form domain differs, change that one line.
- Push `index.html` + `success.html`, enable **Settings → Pages**.

---

## Secrets/vars cheat-sheet

| Where | Name | Value |
|-------|------|-------|
| Worker secret | `STRIPE_SECRET_KEY` | `sk_live_...` |
| Worker secret | `STRIPE_WEBHOOK_SECRET` | `whsec_...` |
| Worker secret | `SHEET_WEBHOOK_URL` | Apps Script web-app URL |
| Worker secret | `SHEET_TOKEN` | random password (also in `Code.gs`) |
| Worker var | `SITE_URL` | your GitHub Pages shop URL |
| Worker binding | `LOGOS` | R2 bucket `exposureboost-logos` |

## Prices (edit in `worker/worker.js` → `PRODUCTS`, in pence)

| ID | Product | Price |
|----|---------|-------|
| `metal` | Metal NFC Card | £99 |
| `wooden` | Wooden NFC Card | £70 |
| `plastic` | Plastic NFC Card | £70 |
| `stand` | NFC Stand | **£25 (placeholder — set the real price)** |

Change a price in **both** `worker/worker.js` and the display values in `index.html`.

## Testing
- Use Stripe **test mode** (`sk_test_...`), card `4242 4242 4242 4242`, any future date/CVC.
- Run the order form locally: `cd worker && npx wrangler dev` → open the printed URL
  with `?cart=metal:1` on the end.
- After a test payment, check the Stripe webhook delivered (Dashboard → Webhooks)
  and that a row landed in your sheet.
