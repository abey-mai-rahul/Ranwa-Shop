# Khata Bahi — Digital Ledger

A mobile-first, installable web app that turns your paper khata into a simple digital ledger. Built as three plain files — no build step, no framework, no bundler — so it can be opened directly, hosted anywhere, or dropped into any static site host.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app: markup, styles, and logic in one file. |
| `manifest.json` | Makes the app installable on Android/iOS home screens. |
| `sw.js` | A small service worker that caches the app shell for offline loading. |

## Running it locally

No installation needed. Two options:

1. **Just open it** — double-click `index.html`. Everything works except the service worker (browsers block service workers on `file://`).
2. **Serve it properly** (recommended, gives you the installable/offline behavior):
   ```bash
   npx serve .
   # or
   python3 -m http.server 8080
   ```
   Then visit the printed local address on your phone or computer.

## Connecting your n8n webhooks (this is where the "backend" lives)

There is no server bundled with this app on purpose — every read and write goes through a webhook you control in n8n. All of them are listed in one place near the top of `index.html`:

```js
const CONFIG = {
  webhooks: {
    getCustomers:        "https://REPLACE_ME.n8n.cloud/webhook/get-customers",
    getCustomer:         "https://REPLACE_ME.n8n.cloud/webhook/get-customer",
    createCustomer:      "https://REPLACE_ME.n8n.cloud/webhook/create-customer",
    deleteCustomer:      "https://REPLACE_ME.n8n.cloud/webhook/delete-customer",
    createTransaction:   "https://REPLACE_ME.n8n.cloud/webhook/create-transaction",
    editTransaction:     "https://REPLACE_ME.n8n.cloud/webhook/edit-transaction",
    deleteTransaction:   "https://REPLACE_ME.n8n.cloud/webhook/delete-transaction",
    getRecentTransactions: "https://REPLACE_ME.n8n.cloud/webhook/recent-transactions",
  },
  storeName: "My Shop",
  useMockData: true
};
```

You can edit these two ways:

- **In code** — replace each URL, then set `useMockData: false`.
- **From the app** — open **Settings → n8n Webhooks**, tap the pencil next to any entry, paste your webhook URL, and save. Toggle **Use Mock Data** off once everything is wired up. (These edits live only for the current browser session — bake the real URLs into the file for a permanent setup.)

### What each webhook needs to do

Every webhook receives a `POST` with a JSON body and must return JSON. Build one n8n workflow per webhook (a Webhook node → your logic, e.g. a database/Google Sheets node → a Respond to Webhook node).

| Function | Request body | Expected response |
|---|---|---|
| `getCustomers` | `{}` | `{ ok: true, data: Customer[] }` |
| `getCustomer` | `{ customerId }` | `{ ok: true, data: { customer: Customer, transactions: Transaction[] } }` |
| `createCustomer` | `{ name, profession, baththa, phone }` | `{ ok: true, data: Customer }` |
| `deleteCustomer` | `{ customerId }` | `{ ok: true }` |
| `createTransaction` | `{ customerId, customerName, amount, description, timestamp }` | `{ ok: true, data: Transaction }` |
| `editTransaction` | `{ id, amount, description }` | `{ ok: true, data: Transaction }` |
| `deleteTransaction` | `{ id }` | `{ ok: true }` |
| `getRecentTransactions` | `{ limit }` | `{ ok: true, data: Transaction[] }` |

```ts
interface Customer {
  id: string; name: string; profession?: string; baththa?: string;
  phone?: string; balance: number; createdAt: string; lastUsed: string;
}
interface Transaction {
  id: string; customerId: string; customerName: string;
  amount: number; description?: string; timestamp: string;
}
```

Notes:
- `amount` is signed: positive means the customer owes more, negative means they paid. The app never asks the user to pick "debit" or "credit" — it reads the sign.
- Keep balance math (adding/subtracting `amount` from the customer's running balance) inside your n8n workflow so it's the single source of truth, matching what the front end also assumes when showing "balance after" on each row.
- Real network errors, a non-200 response, or an unreachable URL are all shown to the shop owner as **"❌ Transaction Failed"** with a **Retry** button — the app never assumes a save succeeded.

### While you don't have n8n set up yet

Leave `useMockData: true` (the default). The app runs entirely on in-memory sample data — a few demo customers and transactions — so you can click through every screen and feature before any backend exists. Nothing is saved between browser refreshes in this mode.

## Installing on an Android phone (PWA)

1. Host the three files somewhere with HTTPS (see Deployment below).
2. Open the URL in Chrome on the phone.
3. Tap the menu (⋮) → **Add to Home screen** (or the automatic "Install app" banner).
4. The app now opens full-screen from the home screen, with the app shell cached for fast, mostly-offline loading. (Creating and viewing data still needs a connection, since that goes through your n8n webhooks.)

## Deployment

Any static host works, since there's no server code to run:

- **Netlify / Vercel** — drag the folder into the dashboard, or connect a git repo and deploy with no build command.
- **GitHub Pages** — push the three files to a repo, enable Pages on the branch.
- **Cloudflare Pages** — same as above, no build command needed.
- **Your own server** — copy the three files into any web root (Nginx, Apache, etc.).

Whichever host you choose, make sure it serves over **HTTPS** — service workers (and therefore installability) require it.

## Customizing

- **Store name** — change it once in Settings, or edit `CONFIG.storeName` in `index.html`.
- **Colors and type** — all design tokens are CSS variables at the top of the `<style>` block (`--paper`, `--ink`, `--marigold`, `--thread`, fonts, etc.) — change once, the whole app updates.
- **Quick amount buttons** — edit the array `[50,100,200,500,1000]` inside `renderNewTransaction()`.

## What's intentionally not included

- No login/authentication — add one at the n8n layer (e.g. an API key header) if this will be exposed publicly.
- No multi-currency support — amounts assume ₹ (INR).
- Share/print statement buttons are present in the UI but marked "coming soon," per the brief.
