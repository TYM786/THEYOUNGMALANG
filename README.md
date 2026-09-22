# THE YOUNG MALANG — POS System

A complete restaurant POS: online/manual/QR/dine-in orders, kitchen display, tables,
delivery + riders, payments, reports, roles — built as plain HTML/CSS/JS (no build step)
on **Supabase** (database + auth + realtime) and deployable free on **Netlify**.

## Files

```
index.html        → POS app (staff: login, dashboard, orders, kitchen, delivery, riders…)
admin.html         → Admin / Management panel (menu, prices, riders, users, settings)
order.html         → Customer website (online ordering + QR table ordering + tracking)
css/app.css        → shared styles (3 themes, responsive, RTL/Urdu support)
js/config.js        → ⚠️ EDIT THIS: your Supabase URL + anon key
js/lang-ur.js       → Urdu translations
js/core.js          → shared: login, theme, language, clock, receipts, modals
js/pos.js, pos-order.js, pos-views.js → POS app logic
js/admin.js          → Admin panel logic
js/order.js          → customer website logic
supabase/schema.sql  → database schema, security rules, business logic (run first)
supabase/seed.sql    → sample menu + tables + settings (run second, edit prices freely after)
assets/logo.jpg, assets/favicon.png → your logo, prepared from the file you sent
```

## 1. Set up Supabase (backend) — free tier is enough to start

1. Go to https://supabase.com → New Project. Pick any name/region, set a database password (save it somewhere).
2. Open **SQL Editor → New query**, paste the **entire contents of `supabase/schema.sql`**, click **Run**.
3. New query again, paste **all of `supabase/seed.sql`**, click **Run**. This adds sample tables (T01–T08),
   a sample menu (edit later from the Admin Panel) and default settings.
4. Go to **Project Settings → API**. Copy the **Project URL** and the **anon public key**.
5. Open `js/config.js` in this folder and paste them in:
   ```js
   SUPABASE_URL: 'https://xxxxxxxx.supabase.co',
   SUPABASE_ANON_KEY: 'eyJ...',
   ```
6. In **Authentication → Providers**, make sure **Email** is enabled (it is by default).
   In **Authentication → Settings**, turn **OFF** "Confirm email" (staff sign up with a
   username, not a real inbox) — otherwise the first Super Admin account can't log in
   until it's confirmed.

## 2. Create your Super Admin (first login)

1. Open `index.html` (double-click it, or after deploying, open your Netlify URL) → click
   **"New staff — create account"** on the login screen.
2. Enter your name, a username (e.g. `admin`, no spaces) and a password (6+ characters).
3. This **first account automatically becomes Super Admin** and is active immediately —
   log straight in. Every account created after that starts **inactive** with the Cashier
   role, until you (as Super Admin, in **Settings → Users** or in `admin.html → Users & Roles`)
   activate it and choose its real role (Manager / Cashier / Kitchen Staff / Delivery Staff).
4. For a Delivery Staff account to clock in/out and see its own deliveries, also add a
   matching Rider in `admin.html → Riders & Incentives` and use **"Link user"** to connect it.

## 3. Edit your menu, prices, riders, tables

Open `admin.html`, log in with your Super Admin account:
- **Menu & Categories** — add/edit products, sizes, add-ons, photos (paste an image URL),
  turn items on/off.
- **Tables** — add or remove dine-in tables; each gets its own QR code automatically
  (`?table=T01` on your order.html link — see below).
- **Riders & Incentives** — add riders, their permanent Rider ID, rate per delivery, and
  bonus rules.
- **Users & Roles** — activate staff accounts and set roles.
- **Business Settings** — restaurant name/phones/address on receipts, delivery fee, tax,
  service charge, payment methods, cashier discount limit, etc.

## 4. QR ordering for tables

Once deployed, each table's QR code should point to:
```
https://youngmalang.com/order.html?table=T01
```
(swap `T01` for each table's code, shown in Admin → Tables). Generate a QR image for each
link with any free QR generator and print it on the table. Scanning it opens the menu
already set to that table — no delivery address needed, and the order appears in POS
tagged **QR** at that table.

## 5. Push to GitHub

```bash
cd young-malang-pos
git init
git add .
git commit -m "THE YOUNG MALANG POS"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/young-malang-pos.git
git push -u origin main
```
(`js/config.js` is committed with your keys in it — that's fine: the anon key is meant to be
public and is safe in a browser; your data is protected by the database security rules
already set up in `schema.sql`.)

## 6. Deploy on Netlify (free)

1. https://app.netlify.com → **Add new site → Import an existing project** → connect your
   GitHub repo.
2. Build command: leave **empty**. Publish directory: `.` (this already matches `netlify.toml`).
3. Deploy. You'll get a `https://something.netlify.app` link — test everything there first.
4. Handy short links once deployed: `/pos` → POS, `/admin` → Admin Panel, `/menu` → customer site.

## 7. Move to youngmalang.com later

Netlify → **Domain settings → Add a custom domain** → `youngmalang.com`, then point your
domain's DNS to Netlify as it instructs (usually just changing the nameservers or adding
an A/CNAME record). No code changes needed — the same `index.html` / `admin.html` /
`order.html` keep working on the new domain.

## Roles at a glance

| Role | Can |
|---|---|
| **Super Admin** | everything, incl. Admin Panel, prices, users, passwords |
| **Manager** | run POS, riders, waiters, tables, cancellations, reports, most settings |
| **Cashier** | create/checkout orders, receive payments, limited discount, no prices/users |
| **Waiter** | "My Tables" — take dine-in orders, get assigned to a table, mark a ready order as Served |
| **Kitchen Staff** | Kitchen Display only — accept, prepare, mark ready |
| **Delivery Staff** | "My Deliveries" — accept/pick up/deliver their own assigned orders |

Every staff member (any role) automatically gets a permanent **Staff ID** (ST-001, ST-002…)
the moment their account is created — it shows on receipts next to their name.

## If you already ran the old schema.sql

`supabase/schema.sql` is safe to run again — it only **adds** the new Waiter role, Staff IDs,
NTN field, Receipt Numbers and the reprint audit log; it never deletes your existing data.
Just paste the whole file into **SQL Editor → New query → Run** again, the same as the first
time. No need to re-run `seed.sql` unless you want the sample menu back.

## What's new in this version

- **Waiter role** — a waiter can log in, see **My Tables**, take a dine-in order for a table
  (or a cashier/manager can assign a waiter when creating the order), and mark it **Served**
  once the kitchen has it ready. Managers see everyone in **Waiters / Staff** with today's
  order count and status (Available / Serving / Offline / Inactive).
- **NTN** — set it once in **Settings → Restaurant & Receipt** (or Admin → Business Settings);
  it then prints on every new receipt automatically.
- **Receipt Number** — every order now also gets its own `RCP-xxxxxx` number, separate from
  the Order Number, shown on every receipt.
- **Served By / Rider on receipts** — dine-in and QR receipts show the assigned waiter; a
  delivery receipt shows the assigned rider and Rider ID; a takeaway/walk-in/phone receipt
  shows the cashier who created it (all with their Staff ID).
- **Receipt print sizes** — the receipt screen now lets you pick 58mm, 80mm, or A4 before
  printing (Download always saves the 80mm-styled version as a digital copy).
- **Reprint audit log** — every print or download is recorded (who, when, which size) in a
  new `audit_logs` table for accountability.
- **Waiter Report** — added to Reports, showing every dine-in order grouped by waiter.

## Notes

- **Real-time**: every screen updates live for every logged-in user (new order → kitchen,
  ready → delivery, rider assigned → riders list) without refreshing.
- **Language / Theme**: buttons on every screen (top bar and login) switch English ⇄ Urdu
  and cycle Classic / Dark / Sunny themes — saved per device.
- **Analog clock**: shown on the login screen and the top bar of the POS.
- **Receipts**: print or download after every order/payment; format matches a narrow
  80mm receipt printer.
- If something ever says *"Supabase is not connected yet"*, it means `js/config.js` still
  has the placeholder `YOUR-PROJECT-ID` — go back to Step 1.
