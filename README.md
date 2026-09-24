# THE YOUNG MALANG — POS System

A complete restaurant POS: online/manual/QR/dine-in orders, kitchen display, tables,
delivery + riders, waiters, payments, reports, roles — built as **4 self-contained files**.
No folders, no build step, no separate CSS/JS files to lose during upload — each HTML file
has its own styling and code built in.

## The 4 files

| File | What it is |
|---|---|
| `index.html` | **POS app** — staff login, dashboard, orders, kitchen, tables, delivery, riders, payments, reports |
| `admin.html` | **Admin / Management panel** — menu, prices, categories, riders, waiters, users, business settings |
| `order.html` | **Customer website** — online ordering + QR table ordering + order tracking (no login needed) |
| `README.md` | This file — setup steps and the database script |

That's it. Nothing else needs to be uploaded.

## 1. Set up the database (Supabase) — do this once

1. Go to https://supabase.com → your project (or create a new one) → **SQL Editor → New query**.
2. Paste **all of the SCHEMA script** below, click **Run**.
3. New query again, paste **all of the SEED script** below, click **Run**. This adds sample
   tables (T01–T08), a sample menu (edit later from the Admin Panel) and default settings.
4. In **Authentication → Settings**, turn **OFF** "Confirm email" (staff sign up with a
   username, not a real inbox).

The Supabase connection details are already filled in inside `index.html`, `admin.html` and
`order.html` — you don't need to edit anything for this part; they already point at:
`https://pmmmnslhseewneswitqa.supabase.co`

If you ever create a **different** Supabase project, open each of the 3 HTML files, find this
near the top of the `<script>` section, and update the two values:
```js
window.YM_CONFIG = {
  SUPABASE_URL: 'https://pmmmnslhseewneswitqa.supabase.co',
  SUPABASE_ANON_KEY: 'sb_publishable_RSPiyMVbHFsU5-_V5MmPKA_GoQdlbl1',
  ...
```

### SCHEMA script (run first)
```sql
-- =====================================================================
-- THE YOUNG MALANG — POS  |  Supabase schema
-- Supabase Dashboard -> SQL Editor -> New query -> paste this whole file -> Run
-- Then run seed.sql (menu + settings + tables).
-- Safe to re-run: uses IF NOT EXISTS / CREATE OR REPLACE where possible.
-- =====================================================================

create extension if not exists pgcrypto with schema extensions;

-- ---------------------------------------------------------------------
-- 1. USERS / ROLES
-- ---------------------------------------------------------------------
create table if not exists public.profiles (
  id            uuid primary key references auth.users(id) on delete cascade,
  username      text unique not null,
  full_name     text not null default '',
  role          text not null default 'cashier',
  is_active     boolean not null default false,
  staff_id      text unique,                 -- e.g. ST-001, auto-assigned to every staff member
  waiter_status text not null default 'offline',   -- only meaningful when role = 'waiter'
  created_at    timestamptz not null default now()
);
alter table public.profiles add column if not exists staff_id text unique;
alter table public.profiles add column if not exists waiter_status text not null default 'offline';
alter table public.profiles drop constraint if exists profiles_role_check;
alter table public.profiles add constraint profiles_role_check
  check (role in ('super_admin','manager','cashier','waiter','kitchen','delivery'));
alter table public.profiles drop constraint if exists profiles_wstatus_check;
alter table public.profiles add constraint profiles_wstatus_check
  check (waiter_status in ('available','serving','offline','inactive'));

create sequence if not exists public.staff_code_seq start 1;

-- backfill: give any already-existing account (created before this update) a Staff ID too
do $$
declare r record;
begin
  for r in select id from public.profiles where staff_id is null order by created_at loop
    update public.profiles set staff_id = 'ST-' || lpad(nextval('public.staff_code_seq')::text, 3, '0') where id = r.id;
  end loop;
end $$;

create or replace function public.ym_role() returns text
language sql stable security definer set search_path = public as $$
  select role from public.profiles where id = auth.uid() and is_active
$$;

create or replace function public.ym_is(roles text[]) returns boolean
language sql stable security definer set search_path = public as $$
  select coalesce(public.ym_role() = any(roles), false)
$$;

-- The very first user who signs up becomes Super Admin. Everyone after that
-- is created INACTIVE until an admin activates them and sets their role.
-- Every profile (any role) automatically gets a permanent Staff ID (ST-001, ST-002…)
-- for receipts ("Served By ... Staff ID: ST-XXX") and audit trails.
create or replace function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = public as $$
declare first_user boolean;
begin
  select not exists (select 1 from public.profiles where role = 'super_admin') into first_user;
  insert into public.profiles (id, username, full_name, role, is_active, staff_id)
  values (new.id,
          lower(split_part(new.email, '@', 1)),
          coalesce(nullif(new.raw_user_meta_data->>'full_name',''), split_part(new.email, '@', 1)),
          case when first_user then 'super_admin' else 'cashier' end,
          first_user,
          'ST-' || lpad(nextval('public.staff_code_seq')::text, 3, '0'));
  return new;
end $$;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function public.handle_new_user();

-- ---------------------------------------------------------------------
-- 2. SETTINGS
-- ---------------------------------------------------------------------
create table if not exists public.settings (
  key        text primary key,
  value      jsonb not null,
  updated_at timestamptz not null default now()
);
-- patch: add the NTN field to an already-existing restaurant settings row (safe to re-run)
update public.settings set value = value || jsonb_build_object('ntn', coalesce(value->>'ntn', ''))
 where key = 'restaurant' and not (value ? 'ntn');

create or replace function public.ym_setting(k text) returns jsonb
language sql stable security definer set search_path = public as $$
  select value from public.settings where key = k
$$;

-- ---------------------------------------------------------------------
-- 3. MENU
-- ---------------------------------------------------------------------
create table if not exists public.categories (
  id        serial primary key,
  name      text not null,
  name_ur   text,
  sort      int not null default 0,
  is_active boolean not null default true
);

create table if not exists public.products (
  id           serial primary key,
  category_id  int references public.categories(id) on delete set null,
  name         text not null,
  name_ur      text,
  description  text,
  price        numeric(10,2) not null default 0,
  image_url    text,
  is_available boolean not null default true,
  is_active    boolean not null default true,
  sort         int not null default 0
);

create table if not exists public.product_variants (
  id         serial primary key,
  product_id int not null references public.products(id) on delete cascade,
  name       text not null,
  name_ur    text,
  price      numeric(10,2) not null,
  sort       int not null default 0
);

create table if not exists public.addons (
  id          serial primary key,
  category_id int references public.categories(id) on delete cascade,  -- null = all categories
  name        text not null,
  name_ur     text,
  price       numeric(10,2) not null default 0,
  is_active   boolean not null default true
);

-- ---------------------------------------------------------------------
-- 4. TABLES (dine-in)
-- ---------------------------------------------------------------------
create table if not exists public.dining_tables (
  id               serial primary key,
  code             text unique not null,
  seats            int not null default 4,
  status           text not null default 'available'
                   check (status in ('available','occupied','reserved')),
  current_order_id bigint,
  qr_code          text generated always as ('QR-' || code) stored
);

-- ---------------------------------------------------------------------
-- 5. CUSTOMERS / ORDERS / PAYMENTS
-- ---------------------------------------------------------------------
create table if not exists public.customers (
  id         bigserial primary key,
  name       text,
  phone      text unique,
  address    text,
  notes      text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create sequence if not exists public.order_no_seq start 1001;
create sequence if not exists public.receipt_no_seq start 4001;

create table if not exists public.orders (
  id             bigserial primary key,
  order_no       text unique not null default ('YM-' || nextval('public.order_no_seq')::text),
  receipt_no     text unique not null default ('RCP-' || lpad(nextval('public.receipt_no_seq')::text, 6, '0')),
  client_ref     uuid unique,                       -- prevents duplicate orders on double click / retry
  source         text not null check (source in ('walkin','dinein','takeaway','phone','online','qr')),
  fulfillment    text not null check (fulfillment in ('dinein','pickup','delivery')),
  status         text not null default 'pending'
                 check (status in ('pending','accepted','preparing','ready','served','assigned',
                                   'picked_up','on_the_way','delivered','completed','cancelled','rejected')),
  table_id       int references public.dining_tables(id),
  waiter_id      uuid references public.profiles(id),      -- who is serving this dine-in / QR table
  customer_id    bigint references public.customers(id),
  customer_name  text,
  phone          text,
  address        text,
  notes          text,
  subtotal       numeric(10,2) not null default 0,
  discount       numeric(10,2) not null default 0,
  delivery_fee   numeric(10,2) not null default 0,
  tax            numeric(10,2) not null default 0,
  service_charge numeric(10,2) not null default 0,
  total          numeric(10,2) not null default 0,
  paid_amount    numeric(10,2) not null default 0,
  payment_status text not null default 'unpaid' check (payment_status in ('unpaid','partial','paid')),
  payment_method text,
  kitchen_batch  int not null default 1,
  cancel_reason  text,
  created_by     uuid references public.profiles(id),
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  confirmed_at   timestamptz,
  ready_at       timestamptz,
  completed_at   timestamptz
);
alter table public.orders add column if not exists receipt_no text;
update public.orders set receipt_no = 'RCP-' || lpad(nextval('public.receipt_no_seq')::text, 6, '0') where receipt_no is null;
alter table public.orders alter column receipt_no set not null;
do $$ begin
  if not exists (select 1 from pg_constraint where conname = 'orders_receipt_no_key') then
    alter table public.orders add constraint orders_receipt_no_key unique (receipt_no);
  end if;
end $$;
alter table public.orders add column if not exists waiter_id uuid references public.profiles(id);
create index if not exists orders_created_idx on public.orders (created_at desc);
create index if not exists orders_status_idx  on public.orders (status);
create index if not exists orders_table_idx   on public.orders (table_id);
create index if not exists orders_phone_idx   on public.orders (phone);
create index if not exists orders_waiter_idx  on public.orders (waiter_id);

create table if not exists public.order_items (
  id         bigserial primary key,
  order_id   bigint not null references public.orders(id) on delete cascade,
  product_id int references public.products(id) on delete set null,
  name       text not null,
  variant    text,
  unit_price numeric(10,2) not null,
  qty        int not null check (qty > 0),
  addons     jsonb not null default '[]',
  note       text,
  line_total numeric(10,2) not null,
  batch      int not null default 1,
  created_at timestamptz not null default now()
);
create index if not exists order_items_order_idx on public.order_items (order_id);

create table if not exists public.payments (
  id          bigserial primary key,
  order_id    bigint not null references public.orders(id) on delete cascade,
  method      text not null,
  amount      numeric(10,2) not null check (amount > 0),
  reference   text,
  client_ref  uuid unique,
  received_by uuid references public.profiles(id),
  created_at  timestamptz not null default now()
);
create index if not exists payments_order_idx   on public.payments (order_id);
create index if not exists payments_created_idx on public.payments (created_at desc);

-- ---------------------------------------------------------------------
-- 6. RIDERS / DELIVERIES / INCENTIVES
-- ---------------------------------------------------------------------
create sequence if not exists public.rider_code_seq start 1;

create table if not exists public.riders (
  id            serial primary key,
  rider_code    text unique not null default ('RDR-' || lpad(nextval('public.rider_code_seq')::text, 3, '0')),
  name          text not null,
  phone         text,
  vehicle_type  text not null default 'Bike',
  vehicle_no    text,
  status        text not null default 'offline'
                check (status in ('available','on_delivery','offline','inactive')),
  user_id       uuid unique references public.profiles(id) on delete set null,
  per_ride_rate numeric(10,2) not null default 100,
  created_at    timestamptz not null default now()
);

create or replace function public.riders_lock_code() returns trigger
language plpgsql as $$
begin
  if new.rider_code is distinct from old.rider_code then
    raise exception 'Rider ID is permanent and cannot be changed';
  end if;
  return new;
end $$;
drop trigger if exists riders_lock_code_t on public.riders;
create trigger riders_lock_code_t before update on public.riders
  for each row execute function public.riders_lock_code();

create table if not exists public.deliveries (
  id             bigserial primary key,
  order_id       bigint not null references public.orders(id) on delete cascade,
  rider_id       int not null references public.riders(id),
  status         text not null default 'assigned'
                 check (status in ('assigned','accepted','picked_up','on_the_way','delivered','cancelled','failed','reassigned')),
  assigned_at    timestamptz not null default now(),
  accepted_at    timestamptz,
  picked_up_at   timestamptz,
  on_the_way_at  timestamptz,
  delivered_at   timestamptz,
  ride_fee       numeric(10,2) not null default 0,
  cod_amount     numeric(10,2) not null default 0,
  cash_collected numeric(10,2) not null default 0,
  cash_submitted numeric(10,2) not null default 0,
  cash_note      text,
  assigned_by    uuid references public.profiles(id)
);
create unique index if not exists one_active_delivery_per_order on public.deliveries (order_id)
  where status in ('assigned','accepted','picked_up','on_the_way');
create unique index if not exists one_active_delivery_per_rider on public.deliveries (rider_id)
  where status in ('assigned','accepted','picked_up','on_the_way');
create index if not exists deliveries_rider_idx on public.deliveries (rider_id, assigned_at desc);

create table if not exists public.rider_incentives (
  id        serial primary key,
  name      text not null,
  period    text not null default 'month' check (period in ('day','week','month','all')),
  target    int not null check (target > 0),
  bonus     numeric(10,2) not null,
  is_active boolean not null default true
);

-- ---------------------------------------------------------------------
-- 6b. AUDIT LOG  (receipt reprints and other tracked actions)
-- ---------------------------------------------------------------------
create table if not exists public.audit_logs (
  id         bigserial primary key,
  actor      uuid references public.profiles(id),
  action     text not null,
  entity     text not null,
  entity_id  text,
  detail     jsonb not null default '{}',
  created_at timestamptz not null default now()
);
create index if not exists audit_logs_entity_idx on public.audit_logs (entity, entity_id);
create index if not exists audit_logs_created_idx on public.audit_logs (created_at desc);

-- ---------------------------------------------------------------------
-- 6c. FUTURE MODULES — scaffolded so Inventory/Purchases can be added
-- later without touching the rest of the database. No screen uses these
-- yet; they exist so the system stays "scalable without rebuilding".
-- ---------------------------------------------------------------------
create table if not exists public.inventory_items (
  id         bigserial primary key,
  name       text not null,
  unit       text not null default 'pcs',
  qty_on_hand numeric(12,2) not null default 0,
  reorder_level numeric(12,2) not null default 0,
  created_at timestamptz not null default now()
);
create table if not exists public.purchases (
  id           bigserial primary key,
  item_id      bigint references public.inventory_items(id),
  supplier     text,
  qty          numeric(12,2) not null,
  cost         numeric(12,2) not null default 0,
  purchased_by uuid references public.profiles(id),
  created_at   timestamptz not null default now()
);

-- =====================================================================
-- 7. INTERNAL HELPERS  (not callable from the browser)
-- =====================================================================

-- Prices every item on the SERVER from the product tables — the browser can never set a price.
create or replace function public.ym_build_items(p_items jsonb) returns jsonb
language plpgsql security definer set search_path = public as $$
declare
  it jsonb; p public.products%rowtype; v public.product_variants%rowtype; a public.addons%rowtype;
  base numeric; addon_total numeric; addon_list jsonb; vname text; q int; result jsonb := '[]'::jsonb;
begin
  if p_items is null or jsonb_typeof(p_items) <> 'array' or jsonb_array_length(p_items) = 0 then
    raise exception 'Order has no items';
  end if;
  if jsonb_array_length(p_items) > 60 then raise exception 'Too many items'; end if;
  for it in select value from jsonb_array_elements(p_items) loop
    select * into p from public.products where id = (it->>'product_id')::int and is_active;
    if not found then raise exception 'Product not found'; end if;
    if not p.is_available then raise exception '% is not available right now', p.name; end if;
    q := greatest(1, least(50, coalesce((it->>'qty')::int, 1)));
    vname := null; base := p.price;
    if nullif(it->>'variant_id','') is not null then
      select * into v from public.product_variants where id = (it->>'variant_id')::int and product_id = p.id;
      if not found then raise exception 'Invalid size for %', p.name; end if;
      base := v.price; vname := v.name;
    elsif exists (select 1 from public.product_variants where product_id = p.id) then
      raise exception 'Please choose a size for %', p.name;
    end if;
    addon_total := 0; addon_list := '[]'::jsonb;
    for a in
      select ad.* from public.addons ad
      where ad.is_active
        and ad.id in (select x::int from jsonb_array_elements_text(coalesce(it->'addon_ids','[]'::jsonb)) as t(x))
        and (ad.category_id is null or ad.category_id = p.category_id)
    loop
      addon_total := addon_total + a.price;
      addon_list := addon_list || jsonb_build_array(jsonb_build_object('name', a.name, 'price', a.price));
    end loop;
    result := result || jsonb_build_array(jsonb_build_object(
      'product_id', p.id, 'name', p.name, 'variant', vname,
      'unit_price', base + addon_total, 'qty', q, 'addons', addon_list,
      'note', left(coalesce(it->>'note',''), 200), 'line_total', (base + addon_total) * q));
  end loop;
  return result;
end $$;

-- Recalculates subtotal, delivery, tax, service charge, total and payment status of an order.
create or replace function public.ym_recalc(p_order bigint) returns void
language plpgsql security definer set search_path = public as $$
declare o public.orders%rowtype; sub numeric; ch jsonb; d numeric; dfee numeric := 0; tx numeric; sc numeric := 0; net numeric; tot numeric;
begin
  select * into o from public.orders where id = p_order;
  select coalesce(sum(line_total), 0) into sub from public.order_items where order_id = p_order;
  ch := coalesce(public.ym_setting('charges'), '{}'::jsonb);
  d := least(o.discount, sub);
  net := sub - d;
  if o.fulfillment = 'delivery' then
    dfee := coalesce((ch->>'delivery_fee')::numeric, 0);
    if coalesce((ch->>'free_delivery_above')::numeric, 0) > 0 and sub >= (ch->>'free_delivery_above')::numeric then dfee := 0; end if;
  end if;
  tx := round(net * coalesce((ch->>'tax_pct')::numeric, 0) / 100);
  if o.fulfillment = 'dinein' then
    sc := round(net * coalesce((ch->>'service_charge_pct')::numeric, 0) / 100);
  end if;
  tot := net + dfee + tx + sc;
  update public.orders set
    subtotal = sub, discount = d, delivery_fee = dfee, tax = tx, service_charge = sc, total = tot,
    payment_status = case when tot <= 0 or paid_amount >= tot then 'paid'
                          when paid_amount > 0 then 'partial' else 'unpaid' end,
    updated_at = now()
  where id = p_order;
end $$;

-- Adds a payment (amount is capped to what is still due). Caller must hold the order lock.
create or replace function public.ym_add_payment(p_order bigint, p_method text, p_amount numeric, p_ref text, p_cref uuid)
returns numeric language plpgsql security definer set search_path = public as $$
declare o public.orders%rowtype; due numeric; amt numeric;
begin
  select * into o from public.orders where id = p_order for update;
  due := o.total - o.paid_amount;
  if due <= 0 then raise exception 'Order is already fully paid'; end if;
  amt := least(round(p_amount, 2), due);
  if amt <= 0 then raise exception 'Invalid amount'; end if;
  insert into public.payments (order_id, method, amount, reference, client_ref, received_by)
  values (p_order, p_method, amt, p_ref, p_cref, auth.uid());
  update public.orders set paid_amount = paid_amount + amt,
    payment_method = case when payment_method is null or payment_method = p_method then p_method else 'Mixed' end
  where id = p_order;
  perform public.ym_recalc(p_order);
  return amt;
end $$;

create or replace function public.ym_occupy_table(p_table int, p_order bigint) returns void
language plpgsql security definer set search_path = public as $$
begin
  if p_table is null then return; end if;
  update public.dining_tables set status = 'occupied', current_order_id = coalesce(current_order_id, p_order) where id = p_table;
end $$;

create or replace function public.ym_release_table(p_table int) returns void
language plpgsql security definer set search_path = public as $$
declare nxt bigint;
begin
  if p_table is null then return; end if;
  select id into nxt from public.orders
   where table_id = p_table and status in ('pending','accepted','preparing','ready','served') order by id limit 1;
  if nxt is null then
    update public.dining_tables set status = 'available', current_order_id = null where id = p_table and status = 'occupied';
  else
    update public.dining_tables set current_order_id = nxt where id = p_table;
  end if;
end $$;

-- =====================================================================
-- 8. STAFF RPCs  (all business actions are atomic + locked + role-checked)
-- =====================================================================

-- ---- create a manual order (walk-in / dine-in / takeaway / phone) ----
create or replace function public.create_order(p jsonb) returns jsonb
language plpgsql security definer set search_path = public as $$
declare
  v_role  text := public.ym_role();
  v_ref   uuid := nullif(p->>'client_ref','')::uuid;
  v_src   text := p->>'source';
  v_ful   text := p->>'fulfillment';
  v_tbl   int  := nullif(p->>'table_id','')::int;
  v_cust  jsonb := coalesce(p->'customer','{}'::jsonb);
  v_name  text := left(nullif(trim(v_cust->>'name'),''), 80);
  v_phone text := nullif(regexp_replace(coalesce(v_cust->>'phone',''), '[^0-9+]', '', 'g'), '');
  v_addr  text := left(nullif(trim(v_cust->>'address'),''), 300);
  v_items jsonb; v_sub numeric; v_disc numeric := 0; v_cid bigint; v_oid bigint;
  v_dtype text := p #>> '{discount,type}';
  v_dval  numeric := coalesce((p #>> '{discount,value}')::numeric, 0);
  v_pay   numeric := coalesce((p #>> '{payment,amount}')::numeric, 0);
  v_method text := nullif(p #>> '{payment,method}', '');
  v_waiter uuid := nullif(p->>'waiter_id','')::uuid;
  v_max numeric; v_tstat text; v_o public.orders%rowtype;
begin
  if v_role is null or v_role not in ('super_admin','manager','cashier','waiter') then raise exception 'Not allowed'; end if;
  if v_ref is not null then
    select * into v_o from public.orders where client_ref = v_ref;
    if found then return jsonb_build_object('id', v_o.id, 'order_no', v_o.order_no, 'total', v_o.total, 'duplicate', true); end if;
  end if;
  if v_src not in ('walkin','dinein','takeaway','phone','online','qr') then raise exception 'Invalid order source'; end if;
  if v_ful not in ('dinein','pickup','delivery') then raise exception 'Invalid order type'; end if;
  if v_role = 'waiter' and v_ful <> 'dinein' then raise exception 'Waiters can only create dine-in orders'; end if;
  if v_ful = 'dinein' then
    if v_tbl is null then raise exception 'Please select a table'; end if;
    select status into v_tstat from public.dining_tables where id = v_tbl for update;
    if not found then raise exception 'Table not found'; end if;
    if v_tstat = 'occupied' then raise exception 'Table already has an order — use Add Items'; end if;
    if v_waiter is null and v_role = 'waiter' then v_waiter := auth.uid(); end if;
    if v_waiter is not null and not exists (select 1 from public.profiles where id = v_waiter and role = 'waiter' and waiter_status <> 'inactive') then
      raise exception 'Selected waiter is not available';
    end if;
  else
    v_tbl := null; v_waiter := null;
  end if;
  if v_ful = 'delivery' and (v_name is null or v_phone is null or v_addr is null) then
    raise exception 'Delivery needs customer name, phone and address';
  end if;
  v_items := public.ym_build_items(p->'items');
  if v_phone is not null then
    insert into public.customers (name, phone, address, notes)
    values (v_name, v_phone, v_addr, nullif(trim(v_cust->>'notes'),''))
    on conflict (phone) do update set
      name = coalesce(excluded.name, public.customers.name),
      address = coalesce(excluded.address, public.customers.address),
      notes = coalesce(excluded.notes, public.customers.notes),
      updated_at = now()
    returning id into v_cid;
  end if;
  insert into public.orders (client_ref, source, fulfillment, status, table_id, waiter_id, customer_id, customer_name, phone, address, notes, created_by, confirmed_at)
  values (v_ref, v_src, v_ful, 'accepted', v_tbl, v_waiter, v_cid, v_name, v_phone, v_addr, left(nullif(trim(p->>'notes'),''), 300), auth.uid(), now())
  returning id into v_oid;
  insert into public.order_items (order_id, product_id, name, variant, unit_price, qty, addons, note, line_total)
  select v_oid, (x->>'product_id')::int, x->>'name', x->>'variant', (x->>'unit_price')::numeric,
         (x->>'qty')::int, x->'addons', nullif(x->>'note',''), (x->>'line_total')::numeric
  from jsonb_array_elements(v_items) x;
  select sum((x->>'line_total')::numeric) into v_sub from jsonb_array_elements(v_items) x;
  if v_dval > 0 then
    v_disc := case when v_dtype = 'percent' then round(v_sub * v_dval / 100) else v_dval end;
    v_disc := least(v_disc, v_sub);
    if v_role = 'cashier' then
      v_max := coalesce((public.ym_setting('permissions')->>'cashier_max_discount_pct')::numeric, 0);
      if v_disc > round(v_sub * v_max / 100) then
        raise exception 'Cashier discount limit is % percent', v_max;
      end if;
    end if;
    update public.orders set discount = v_disc where id = v_oid;
  end if;
  perform public.ym_recalc(v_oid);
  if v_ful = 'dinein' then perform public.ym_occupy_table(v_tbl, v_oid); end if;
  if v_pay > 0 then perform public.ym_add_payment(v_oid, coalesce(v_method, 'Cash'), v_pay, null, null); end if;
  select * into v_o from public.orders where id = v_oid;
  return jsonb_build_object('id', v_o.id, 'order_no', v_o.order_no, 'total', v_o.total);
end $$;

-- ---- add items to an open order (dine-in "Add Items") ----
create or replace function public.add_items(p_order bigint, p_items jsonb) returns jsonb
language plpgsql security definer set search_path = public as $$
declare v_o public.orders%rowtype; v_items jsonb; v_batch int;
begin
  if not public.ym_is(array['super_admin','manager','cashier','waiter']) then raise exception 'Not allowed'; end if;
  select * into v_o from public.orders where id = p_order for update;
  if not found then raise exception 'Order not found'; end if;
  if v_o.status in ('completed','cancelled','rejected','delivered') then raise exception 'Order is closed'; end if;
  if v_o.status in ('assigned','picked_up','on_the_way') then raise exception 'Order is already out for delivery'; end if;
  v_items := public.ym_build_items(p_items);
  v_batch := v_o.kitchen_batch + 1;
  update public.orders set kitchen_batch = v_batch,
         status = case when status in ('ready','served') then 'accepted' else status end
   where id = p_order;
  insert into public.order_items (order_id, product_id, name, variant, unit_price, qty, addons, note, line_total, batch)
  select p_order, (x->>'product_id')::int, x->>'name', x->>'variant', (x->>'unit_price')::numeric,
         (x->>'qty')::int, x->'addons', nullif(x->>'note',''), (x->>'line_total')::numeric, v_batch
  from jsonb_array_elements(v_items) x;
  perform public.ym_recalc(p_order);
  return jsonb_build_object('id', p_order, 'order_no', v_o.order_no);
end $$;

-- ---- change order status (accept / reject / preparing / ready / served / completed) ----
create or replace function public.set_order_status(p_order bigint, p_status text, p_reason text default null)
returns void language plpgsql security definer set search_path = public as $$
declare v_role text := public.ym_role(); o public.orders%rowtype; ok boolean := false;
begin
  if v_role is null then raise exception 'Not allowed'; end if;
  select * into o from public.orders where id = p_order for update;
  if not found then raise exception 'Order not found'; end if;
  if o.status = p_status then return; end if;   -- double click safe
  if v_role = 'kitchen' then
    if not ((o.status = 'accepted' and p_status = 'preparing') or (o.status = 'preparing' and p_status = 'ready')) then
      raise exception 'Not allowed';
    end if;
  elsif v_role in ('super_admin','manager','cashier') then
    ok := (o.status = 'pending'   and p_status in ('accepted','rejected'))
       or (o.status = 'accepted'  and p_status in ('preparing','ready'))
       or (o.status = 'preparing' and p_status = 'ready')
       or (o.status = 'ready'     and p_status in ('served','completed'))
       or (o.status = 'served'    and p_status = 'completed');
    if not ok then raise exception 'Cannot change status from % to %', o.status, p_status; end if;
    if p_status in ('served','completed') and o.fulfillment = 'delivery' then
      raise exception 'Delivery orders are completed by the rider';
    end if;
    if p_status = 'served' and o.fulfillment <> 'dinein' then raise exception 'Only dine-in orders can be served'; end if;
    if p_status = 'completed' and o.payment_status <> 'paid' then raise exception 'Receive payment first'; end if;
  elsif v_role = 'waiter' then
    if not (o.status = 'ready' and p_status = 'served' and o.fulfillment = 'dinein' and o.waiter_id = auth.uid()) then
      raise exception 'Not allowed';
    end if;
  else
    raise exception 'Not allowed';
  end if;

  update public.orders set status = p_status,
    cancel_reason = case when p_status = 'rejected' then p_reason else cancel_reason end,
    confirmed_at = case when p_status = 'accepted' and confirmed_at is null then now() else confirmed_at end,
    ready_at = case when p_status = 'ready' then now() else ready_at end,
    completed_at = case when p_status = 'completed' then now() else completed_at end,
    updated_at = now()
  where id = p_order;

  if p_status = 'accepted' and o.status = 'pending' and o.phone is not null then
    insert into public.customers (name, phone, address) values (o.customer_name, o.phone, o.address)
    on conflict (phone) do nothing;
    update public.orders set customer_id = (select id from public.customers where phone = o.phone) where id = p_order;
  end if;
  if p_status in ('completed','rejected') then perform public.ym_release_table(o.table_id); end if;
end $$;

-- ---- cancel ----
create or replace function public.cancel_order(p_order bigint, p_reason text) returns void
language plpgsql security definer set search_path = public as $$
declare v_role text := public.ym_role(); o public.orders%rowtype; can_cancel boolean;
begin
  if v_role is null or v_role not in ('super_admin','manager','cashier') then raise exception 'Not allowed'; end if;
  select * into o from public.orders where id = p_order for update;
  if not found then raise exception 'Order not found'; end if;
  if o.status in ('completed','cancelled','rejected','delivered') then raise exception 'Order is already closed'; end if;
  if v_role = 'cashier' then
    can_cancel := coalesce((public.ym_setting('permissions')->>'cashier_can_cancel')::boolean, false)
                  and o.status in ('pending','accepted');
    if not can_cancel then raise exception 'Only a manager can cancel this order'; end if;
  end if;
  update public.riders set status = 'available'
   where id in (select rider_id from public.deliveries
                where order_id = p_order and status in ('assigned','accepted','picked_up','on_the_way'))
     and status = 'on_delivery';
  update public.deliveries set status = 'cancelled'
   where order_id = p_order and status in ('assigned','accepted','picked_up','on_the_way');
  update public.orders set status = 'cancelled', cancel_reason = left(p_reason, 200), updated_at = now() where id = p_order;
  perform public.ym_release_table(o.table_id);
end $$;

-- ---- record payment ----
create or replace function public.record_payment(p_order bigint, p_method text, p_amount numeric, p_ref text default null, p_cref uuid default null)
returns jsonb language plpgsql security definer set search_path = public as $$
declare o public.orders%rowtype; amt numeric;
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  if p_cref is not null and exists (select 1 from public.payments where client_ref = p_cref) then
    return jsonb_build_object('duplicate', true);
  end if;
  select * into o from public.orders where id = p_order for update;
  if not found then raise exception 'Order not found'; end if;
  if o.status in ('cancelled','rejected') then raise exception 'Order is cancelled'; end if;
  amt := public.ym_add_payment(p_order, p_method, p_amount, p_ref, p_cref);
  return jsonb_build_object('amount', amt);
end $$;

-- ---- close table ----
create or replace function public.close_table(p_table int) returns void
language plpgsql security definer set search_path = public as $$
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  perform 1 from public.dining_tables where id = p_table for update;
  perform 1 from public.orders where table_id = p_table and status in ('pending','accepted','preparing','ready','served') for update;
  if exists (select 1 from public.orders where table_id = p_table and status in ('pending','accepted','preparing')) then
    raise exception 'Kitchen is still working on an order for this table';
  end if;
  if exists (select 1 from public.orders where table_id = p_table and status in ('ready','served') and payment_status <> 'paid') then
    raise exception 'Receive full payment before closing the table';
  end if;
  update public.orders set status = 'completed', completed_at = now(), updated_at = now()
   where table_id = p_table and status in ('ready','served');
  update public.dining_tables set status = 'available', current_order_id = null where id = p_table;
end $$;

create or replace function public.set_table_status(p_table int, p_status text) returns void
language plpgsql security definer set search_path = public as $$
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  if p_status not in ('available','reserved') then raise exception 'Invalid status'; end if;
  perform 1 from public.dining_tables where id = p_table for update;
  if exists (select 1 from public.orders where table_id = p_table and status in ('pending','accepted','preparing','ready','served')) then
    raise exception 'Table has an open order';
  end if;
  update public.dining_tables set status = p_status, current_order_id = null where id = p_table;
end $$;

create or replace function public.set_product_available(p_product int, p_flag boolean) returns void
language plpgsql security definer set search_path = public as $$
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  update public.products set is_available = p_flag where id = p_product;
end $$;

-- =====================================================================
-- 9. RIDER / DELIVERY RPCs
-- =====================================================================

create or replace function public.assign_rider(p_order bigint, p_rider int) returns bigint
language plpgsql security definer set search_path = public as $$
declare o public.orders%rowtype; r public.riders%rowtype; v_id bigint;
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  select * into o from public.orders where id = p_order for update;
  if not found then raise exception 'Order not found'; end if;
  if o.fulfillment <> 'delivery' then raise exception 'This is not a delivery order'; end if;
  if o.status <> 'ready' then raise exception 'Order must be READY before assigning a rider'; end if;
  select * into r from public.riders where id = p_rider for update;
  if not found then raise exception 'Rider not found'; end if;
  if r.status <> 'available' then raise exception '% is not available', r.name; end if;
  insert into public.deliveries (order_id, rider_id, ride_fee, cod_amount, assigned_by)
  values (p_order, p_rider, r.per_ride_rate, greatest(o.total - o.paid_amount, 0), auth.uid())
  returning id into v_id;
  update public.riders set status = 'on_delivery' where id = p_rider;
  update public.orders set status = 'assigned', updated_at = now() where id = p_order;
  return v_id;
end $$;

create or replace function public.update_delivery_status(p_delivery bigint, p_status text, p_cash numeric default null)
returns void language plpgsql security definer set search_path = public as $$
declare
  v_role text := public.ym_role(); v_oid bigint;
  d public.deliveries%rowtype; r public.riders%rowtype; o public.orders%rowtype;
  ok boolean; due numeric; amt numeric := 0;
begin
  if v_role is null then raise exception 'Not allowed'; end if;
  select order_id into v_oid from public.deliveries where id = p_delivery;
  if not found then raise exception 'Delivery not found'; end if;
  select * into o from public.orders where id = v_oid for update;
  select * into d from public.deliveries where id = p_delivery for update;
  select * into r from public.riders where id = d.rider_id for update;
  if v_role = 'delivery' then
    if r.user_id is distinct from auth.uid() then raise exception 'This delivery is not assigned to you'; end if;
  elsif v_role not in ('super_admin','manager','cashier') then
    raise exception 'Not allowed';
  end if;
  if d.status = p_status then return; end if;  -- double click safe
  ok := (d.status = 'assigned'   and p_status in ('accepted','picked_up'))
     or (d.status = 'accepted'   and p_status = 'picked_up')
     or (d.status = 'picked_up'  and p_status in ('on_the_way','delivered'))
     or (d.status = 'on_the_way' and p_status = 'delivered')
     or (d.status in ('assigned','accepted','picked_up','on_the_way') and p_status = 'failed');
  if not ok then raise exception 'Delivery is already % — cannot change to %', d.status, p_status; end if;

  if p_status = 'delivered' then
    due := greatest(o.total - o.paid_amount, 0);
    if due > 0 then
      amt := least(coalesce(p_cash, due), due);
      if amt > 0 then perform public.ym_add_payment(o.id, 'Cash', amt, 'COD ' || r.rider_code, null); end if;
    end if;
    update public.deliveries set status = 'delivered', delivered_at = now(), ride_fee = r.per_ride_rate,
           cash_collected = amt, cod_amount = due where id = p_delivery;
    update public.orders set status = 'delivered', completed_at = now(), updated_at = now() where id = o.id;
    update public.riders set status = 'available' where id = r.id and status = 'on_delivery';
  elsif p_status = 'failed' then
    update public.deliveries set status = 'failed' where id = p_delivery;
    update public.orders set status = 'ready', updated_at = now() where id = o.id;
    update public.riders set status = 'available' where id = r.id and status = 'on_delivery';
  else
    update public.deliveries set status = p_status,
      accepted_at   = case when p_status = 'accepted'   then now() else accepted_at end,
      picked_up_at  = case when p_status = 'picked_up'  then now() else picked_up_at end,
      on_the_way_at = case when p_status = 'on_the_way' then now() else on_the_way_at end
    where id = p_delivery;
    if p_status in ('picked_up','on_the_way') then
      update public.orders set status = p_status, updated_at = now() where id = o.id;
    end if;
  end if;
end $$;

create or replace function public.reassign_rider(p_delivery bigint, p_rider int) returns bigint
language plpgsql security definer set search_path = public as $$
declare d public.deliveries%rowtype; o public.orders%rowtype; nr public.riders%rowtype; v_oid bigint; v_new bigint;
begin
  if not public.ym_is(array['super_admin','manager']) then raise exception 'Only a manager can reassign riders'; end if;
  select order_id into v_oid from public.deliveries where id = p_delivery;
  if not found then raise exception 'Delivery not found'; end if;
  select * into o from public.orders where id = v_oid for update;
  select * into d from public.deliveries where id = p_delivery for update;
  if d.status not in ('assigned','accepted','picked_up','on_the_way') then raise exception 'Delivery is no longer active'; end if;
  if d.rider_id = p_rider then raise exception 'Choose a different rider'; end if;
  perform 1 from public.riders where id in (d.rider_id, p_rider) order by id for update;
  select * into nr from public.riders where id = p_rider;
  if not found or nr.status <> 'available' then raise exception 'Selected rider is not available'; end if;
  update public.deliveries set status = 'reassigned' where id = p_delivery;
  update public.riders set status = 'available' where id = d.rider_id and status = 'on_delivery';
  insert into public.deliveries (order_id, rider_id, ride_fee, cod_amount, assigned_by)
  values (o.id, p_rider, nr.per_ride_rate, greatest(o.total - o.paid_amount, 0), auth.uid()) returning id into v_new;
  update public.riders set status = 'on_delivery' where id = p_rider;
  update public.orders set status = 'assigned', updated_at = now() where id = o.id;
  return v_new;
end $$;

create or replace function public.set_rider_status(p_rider int, p_status text) returns void
language plpgsql security definer set search_path = public as $$
declare v_role text := public.ym_role(); r public.riders%rowtype;
begin
  if v_role is null then raise exception 'Not allowed'; end if;
  if p_status not in ('available','offline','inactive') then raise exception 'Invalid status'; end if;
  select * into r from public.riders where id = p_rider for update;
  if not found then raise exception 'Rider not found'; end if;
  if v_role = 'delivery' then
    if r.user_id is distinct from auth.uid() or p_status = 'inactive' then raise exception 'Not allowed'; end if;
  elsif v_role not in ('super_admin','manager') then
    raise exception 'Not allowed';
  end if;
  if r.status = 'on_delivery' then raise exception 'Rider is on a delivery'; end if;
  if r.status = 'inactive' and v_role not in ('super_admin','manager') then raise exception 'Rider is disabled by management'; end if;
  update public.riders set status = p_status where id = p_rider;
end $$;

create or replace function public.submit_rider_cash(p_delivery bigint, p_amount numeric, p_note text default null)
returns void language plpgsql security definer set search_path = public as $$
begin
  if not public.ym_is(array['super_admin','manager','cashier']) then raise exception 'Not allowed'; end if;
  if p_amount is null or p_amount <= 0 then raise exception 'Invalid amount'; end if;
  update public.deliveries set cash_submitted = cash_submitted + p_amount,
         cash_note = coalesce(nullif(left(p_note, 200), ''), cash_note)
   where id = p_delivery;
end $$;

-- =====================================================================
-- 9b. WAITER RPC
-- =====================================================================
create or replace function public.set_waiter_status(p_user uuid, p_status text) returns void
language plpgsql security definer set search_path = public as $$
declare v_role text := public.ym_role();
begin
  if v_role is null then raise exception 'Not allowed'; end if;
  if p_status not in ('available','serving','offline','inactive') then raise exception 'Invalid status'; end if;
  if v_role = 'waiter' then
    if p_user <> auth.uid() or p_status = 'inactive' then raise exception 'Not allowed'; end if;
  elsif v_role not in ('super_admin','manager') then
    raise exception 'Not allowed';
  end if;
  update public.profiles set waiter_status = p_status where id = p_user and role = 'waiter';
end $$;

-- Reprints are logged here so every print/download is auditable (spec: "every reprint audit log save ho").
create or replace function public.log_receipt_print(p_order bigint, p_method text) returns void
language plpgsql security definer set search_path = public as $$
begin
  if public.ym_role() is null then raise exception 'Not allowed'; end if;
  insert into public.audit_logs (actor, action, entity, entity_id, detail)
  values (auth.uid(), 'receipt_print', 'order', p_order::text, jsonb_build_object('method', p_method));
end $$;

-- =====================================================================
-- 10. ADMIN RPC
-- =====================================================================
create or replace function public.admin_set_password(p_user uuid, p_password text) returns void
language plpgsql security definer set search_path = public, extensions, auth as $$
begin
  if not public.ym_is(array['super_admin']) then raise exception 'Only Super Admin can do this'; end if;
  if length(coalesce(p_password,'')) < 6 then raise exception 'Password / PIN must be at least 6 characters'; end if;
  update auth.users set encrypted_password = crypt(p_password, gen_salt('bf')), updated_at = now() where id = p_user;
end $$;

-- =====================================================================
-- 11. PUBLIC (customer website / QR) RPCs — callable without login
-- =====================================================================
create or replace function public.place_online_order(p jsonb) returns jsonb
language plpgsql security definer set search_path = public as $$
declare
  v_ref   uuid := nullif(p->>'client_ref','')::uuid;
  v_ful   text := p->>'fulfillment';
  v_code  text := nullif(trim(p->>'table_code'),'');
  v_src   text; v_tbl int;
  v_name  text := left(nullif(trim(p #>> '{customer,name}'),''), 80);
  v_phone text := nullif(regexp_replace(coalesce(p #>> '{customer,phone}',''), '[^0-9+]', '', 'g'), '');
  v_addr  text := left(nullif(trim(p #>> '{customer,address}'),''), 300);
  v_items jsonb; v_oid bigint; v_o public.orders%rowtype; v_min numeric; v_sub numeric;
  v_on jsonb := coalesce(public.ym_setting('online'), '{}'::jsonb);
begin
  if coalesce((v_on->>'accepting_orders')::boolean, true) = false then
    raise exception 'Online ordering is paused right now. Please call us.';
  end if;
  if v_ref is not null then
    select * into v_o from public.orders where client_ref = v_ref;
    if found then return jsonb_build_object('id', v_o.id, 'order_no', v_o.order_no, 'total', v_o.total, 'status', v_o.status, 'duplicate', true); end if;
  end if;
  if v_code is not null then
    v_src := 'qr'; v_ful := 'dinein';
    select id into v_tbl from public.dining_tables where code = upper(v_code);
    if not found then raise exception 'Table not found'; end if;
  else
    v_src := 'online'; v_tbl := null;
    if v_ful not in ('delivery','pickup') then raise exception 'Invalid order type'; end if;
  end if;
  if v_name is null then raise exception 'Please enter your name'; end if;
  if v_src = 'online' and (v_phone is null or length(v_phone) < 10) then raise exception 'Please enter a valid phone number'; end if;
  if v_ful = 'delivery' and v_addr is null then raise exception 'Please enter your delivery address'; end if;
  if v_phone is not null and (select count(*) from public.orders
        where phone = v_phone and status = 'pending' and created_at > now() - interval '15 minutes') >= 3 then
    raise exception 'Too many pending orders. Please wait for confirmation or call us.';
  end if;
  v_items := public.ym_build_items(p->'items');
  select sum((x->>'line_total')::numeric) into v_sub from jsonb_array_elements(v_items) x;
  v_min := coalesce((public.ym_setting('charges')->>'min_online_order')::numeric, 0);
  if v_src = 'online' and v_sub < v_min then raise exception 'Minimum order is Rs. %', v_min; end if;
  insert into public.orders (client_ref, source, fulfillment, status, table_id, customer_name, phone, address, notes)
  values (v_ref, v_src, v_ful, 'pending', v_tbl, v_name, v_phone, v_addr, left(nullif(trim(p->>'notes'),''), 300))
  returning id into v_oid;
  insert into public.order_items (order_id, product_id, name, variant, unit_price, qty, addons, note, line_total)
  select v_oid, (x->>'product_id')::int, x->>'name', x->>'variant', (x->>'unit_price')::numeric,
         (x->>'qty')::int, x->'addons', nullif(x->>'note',''), (x->>'line_total')::numeric
  from jsonb_array_elements(v_items) x;
  perform public.ym_recalc(v_oid);
  if v_tbl is not null then perform public.ym_occupy_table(v_tbl, v_oid); end if;
  select * into v_o from public.orders where id = v_oid;
  return jsonb_build_object('id', v_o.id, 'order_no', v_o.order_no, 'total', v_o.total, 'status', v_o.status);
end $$;

create or replace function public.track_order(p_order_no text, p_phone text default null) returns jsonb
language plpgsql security definer set search_path = public as $$
declare o public.orders%rowtype; v_phone text := nullif(regexp_replace(coalesce(p_phone,''), '[^0-9+]', '', 'g'), ''); v_rider text;
begin
  select * into o from public.orders where order_no = upper(trim(p_order_no));
  if not found then return null; end if;
  if not ((o.phone is not null and o.phone = v_phone) or o.source = 'qr') then return null; end if;
  select r.name into v_rider from public.deliveries d join public.riders r on r.id = d.rider_id
   where d.order_id = o.id and d.status in ('assigned','accepted','picked_up','on_the_way','delivered') order by d.id desc limit 1;
  return jsonb_build_object('order_no', o.order_no, 'status', o.status, 'fulfillment', o.fulfillment,
                            'total', o.total, 'created_at', o.created_at, 'rider', v_rider);
end $$;

-- =====================================================================
-- 12. ROW LEVEL SECURITY
-- =====================================================================
alter table public.profiles         enable row level security;
alter table public.settings         enable row level security;
alter table public.categories       enable row level security;
alter table public.products         enable row level security;
alter table public.product_variants enable row level security;
alter table public.addons           enable row level security;
alter table public.dining_tables    enable row level security;
alter table public.customers        enable row level security;
alter table public.orders           enable row level security;
alter table public.order_items      enable row level security;
alter table public.payments         enable row level security;
alter table public.riders           enable row level security;
alter table public.deliveries       enable row level security;
alter table public.rider_incentives enable row level security;
alter table public.audit_logs       enable row level security;
alter table public.inventory_items  enable row level security;
alter table public.purchases        enable row level security;

do $$
declare r record;
begin
  for r in select schemaname, tablename, policyname from pg_policies where schemaname = 'public' loop
    execute format('drop policy %I on %I.%I', r.policyname, r.schemaname, r.tablename);
  end loop;
end $$;

-- profiles
create policy profiles_read   on public.profiles for select to authenticated
  using (id = auth.uid() or public.ym_role() is not null);
create policy profiles_update on public.profiles for update to authenticated
  using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));

-- settings
create policy settings_public on public.settings for select to anon
  using (key in ('restaurant','charges','payment_methods','online'));
create policy settings_read   on public.settings for select to authenticated
  using (public.ym_role() is not null);
create policy settings_admin  on public.settings for all to authenticated
  using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy settings_mgr_ins on public.settings for insert to authenticated
  with check (public.ym_is(array['manager']) and key not in ('permissions','rider'));
create policy settings_mgr_upd on public.settings for update to authenticated
  using (public.ym_is(array['manager']) and key not in ('permissions','rider'))
  with check (public.ym_is(array['manager']) and key not in ('permissions','rider'));

-- menu (public read of active items; only Super Admin edits)
create policy cat_public  on public.categories for select to anon using (is_active);
create policy cat_staff   on public.categories for select to authenticated using (true);
create policy cat_admin   on public.categories for all to authenticated using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy prod_public on public.products for select to anon using (is_active);
create policy prod_staff  on public.products for select to authenticated using (true);
create policy prod_admin  on public.products for all to authenticated using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy var_public  on public.product_variants for select to anon using (true);
create policy var_staff   on public.product_variants for select to authenticated using (true);
create policy var_admin   on public.product_variants for all to authenticated using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy addon_public on public.addons for select to anon using (is_active);
create policy addon_staff  on public.addons for select to authenticated using (true);
create policy addon_admin  on public.addons for all to authenticated using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));

-- tables
create policy tbl_public on public.dining_tables for select to anon using (true);
create policy tbl_staff  on public.dining_tables for select to authenticated using (true);
create policy tbl_admin  on public.dining_tables for all to authenticated using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy tbl_mgr_ins on public.dining_tables for insert to authenticated with check (public.ym_is(array['manager']));

-- customers
create policy cust_staff_r on public.customers for select to authenticated using (public.ym_is(array['super_admin','manager','cashier']));
create policy cust_staff_i on public.customers for insert to authenticated with check (public.ym_is(array['super_admin','manager','cashier']));
create policy cust_staff_u on public.customers for update to authenticated using (public.ym_is(array['super_admin','manager','cashier'])) with check (public.ym_is(array['super_admin','manager','cashier']));

-- orders + items + payments (writes only via RPC)
create policy orders_staff on public.orders for select to authenticated
  using (public.ym_is(array['super_admin','manager','cashier']));
create policy orders_kitchen on public.orders for select to authenticated
  using (public.ym_is(array['kitchen']) and status not in ('pending','cancelled','rejected'));
create policy orders_rider on public.orders for select to authenticated
  using (public.ym_is(array['delivery']) and exists (
    select 1 from public.deliveries d join public.riders r on r.id = d.rider_id
    where d.order_id = orders.id and r.user_id = auth.uid()));
create policy orders_waiter on public.orders for select to authenticated
  using (public.ym_is(array['waiter']) and (waiter_id = auth.uid() or created_by = auth.uid()));
create policy items_read on public.order_items for select to authenticated
  using (exists (select 1 from public.orders o where o.id = order_items.order_id));
create policy pay_staff on public.payments for select to authenticated
  using (public.ym_is(array['super_admin','manager','cashier']));

-- riders / deliveries / incentives
create policy riders_staff on public.riders for select to authenticated
  using (public.ym_is(array['super_admin','manager','cashier']) or user_id = auth.uid());
create policy riders_admin on public.riders for all to authenticated
  using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));
create policy deliv_staff on public.deliveries for select to authenticated
  using (public.ym_is(array['super_admin','manager','cashier'])
         or rider_id in (select id from public.riders where user_id = auth.uid()));
create policy incent_read on public.rider_incentives for select to authenticated
  using (public.ym_is(array['super_admin','manager','delivery']));
create policy incent_admin on public.rider_incentives for all to authenticated
  using (public.ym_is(array['super_admin'])) with check (public.ym_is(array['super_admin']));

-- audit log + future modules (management only)
create policy audit_read on public.audit_logs for select to authenticated
  using (public.ym_is(array['super_admin','manager']));
create policy audit_insert on public.audit_logs for insert to authenticated
  with check (public.ym_role() is not null and actor = auth.uid());
create policy inv_admin on public.inventory_items for all to authenticated
  using (public.ym_is(array['super_admin','manager'])) with check (public.ym_is(array['super_admin','manager']));
create policy purch_admin on public.purchases for all to authenticated
  using (public.ym_is(array['super_admin','manager'])) with check (public.ym_is(array['super_admin','manager']));

-- =====================================================================
-- 13. FUNCTION PERMISSIONS
-- =====================================================================
revoke all on all functions in schema public from public, anon, authenticated;
grant execute on function public.ym_role(), public.ym_is(text[]) to anon, authenticated;
grant execute on function
  public.create_order(jsonb), public.add_items(bigint, jsonb), public.set_order_status(bigint, text, text),
  public.cancel_order(bigint, text), public.record_payment(bigint, text, numeric, text, uuid),
  public.close_table(int), public.set_table_status(int, text), public.set_product_available(int, boolean),
  public.assign_rider(bigint, int), public.update_delivery_status(bigint, text, numeric),
  public.reassign_rider(bigint, int), public.set_rider_status(int, text),
  public.submit_rider_cash(bigint, numeric, text), public.admin_set_password(uuid, text),
  public.set_waiter_status(uuid, text), public.log_receipt_print(bigint, text)
  to authenticated;
grant execute on function public.place_online_order(jsonb), public.track_order(text, text) to anon, authenticated;

-- =====================================================================
-- 14. REAL-TIME
-- =====================================================================
do $$
declare t text;
begin
  foreach t in array array['orders','order_items','payments','riders','deliveries','dining_tables','products','settings','profiles'] loop
    begin
      execute format('alter publication supabase_realtime add table public.%I', t);
    exception when duplicate_object then null;
    end;
  end loop;
end $$;

-- =====================================================================
-- 15. STORAGE (product photos)
-- =====================================================================
insert into storage.buckets (id, name, public) values ('menu', 'menu', true) on conflict (id) do nothing;
drop policy if exists menu_admin_write on storage.objects;
create policy menu_admin_write on storage.objects for all to authenticated
  using (bucket_id = 'menu' and public.ym_is(array['super_admin']))
  with check (bucket_id = 'menu' and public.ym_is(array['super_admin']));

```

### SEED script (run second)
```sql
-- =====================================================================
-- THE YOUNG MALANG — seed data (run AFTER schema.sql)
-- Prices are SAMPLE prices — change them from the Admin panel (admin.html).
-- Safe to re-run: menu is only inserted when the menu is empty.
-- =====================================================================

insert into public.settings (key, value) values
 ('restaurant', '{"name":"THE YOUNG MALANG","tagline":"Pizza & Burger","phones":["0349-8190030","0337-1111475"],"address":"Near Rajput Marriage Hall, Sardar Market","ntn":"","receipt_footer":"Thank you for visiting The Young Malang!"}'),
 ('charges', '{"delivery_fee":150,"free_delivery_above":0,"tax_pct":0,"service_charge_pct":0,"min_online_order":0}'),
 ('payment_methods', '["Cash","Card","Online Payment","Easypaisa","JazzCash"]'),
 ('permissions', '{"cashier_max_discount_pct":10,"cashier_can_cancel":false}'),
 ('kitchen', '{"sound":true}'),
 ('online', '{"accepting_orders":true}'),
 ('rider', '{"default_rate":100}')
on conflict (key) do nothing;

insert into public.dining_tables (code, seats)
select 'T' || lpad(g::text, 2, '0'), case when g <= 4 then 4 else 6 end
from generate_series(1, 8) g
on conflict (code) do nothing;

insert into public.rider_incentives (name, period, target, bonus)
select * from (values
  ('100 rides this month',  'month', 100, 2000),
  ('200 rides this month',  'month', 200, 5000),
  ('300 rides this month',  'month', 300, 8000),
  ('15 deliveries in a day','day',    15,  500)
) v(name, period, target, bonus)
where not exists (select 1 from public.rider_incentives);

do $$
declare c_pizza int; c_burger int; c_fries int; c_snack int; c_drink int; c_deal int; c_dessert int; pid int;
begin
  if exists (select 1 from public.categories) then return; end if;

  insert into public.categories (name, name_ur, sort) values ('Pizza', 'پیزا', 1) returning id into c_pizza;
  insert into public.categories (name, name_ur, sort) values ('Burgers', 'برگر', 2) returning id into c_burger;
  insert into public.categories (name, name_ur, sort) values ('Fries', 'فرائز', 3) returning id into c_fries;
  insert into public.categories (name, name_ur, sort) values ('Snacks', 'اسنیکس', 4) returning id into c_snack;
  insert into public.categories (name, name_ur, sort) values ('Drinks', 'مشروبات', 5) returning id into c_drink;
  insert into public.categories (name, name_ur, sort) values ('Deals', 'ڈیلز', 6) returning id into c_deal;
  insert into public.categories (name, name_ur, sort) values ('Desserts', 'میٹھا', 7) returning id into c_dessert;

  -- Pizzas (sizes)
  insert into public.products (category_id, name, name_ur, price, sort) values (c_pizza, 'Chicken Tikka Pizza', 'چکن تکہ پیزا', 0, 1) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Small','چھوٹا',550,1),(pid,'Medium','درمیانہ',750,2),(pid,'Large','بڑا',950,3);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_pizza, 'Fajita Pizza', 'فاجیتا پیزا', 0, 2) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Small','چھوٹا',550,1),(pid,'Medium','درمیانہ',750,2),(pid,'Large','بڑا',950,3);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_pizza, 'Cheese Lover Pizza', 'چیز لور پیزا', 0, 3) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Small','چھوٹا',600,1),(pid,'Medium','درمیانہ',800,2),(pid,'Large','بڑا',1000,3);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_pizza, 'Pepperoni Pizza', 'پیپرونی پیزا', 0, 4) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Small','چھوٹا',650,1),(pid,'Medium','درمیانہ',850,2),(pid,'Large','بڑا',1050,3);

  -- Burgers
  insert into public.products (category_id, name, name_ur, price, sort) values (c_burger, 'Zinger Burger', 'زنگر برگر', 0, 1) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Regular','عام',550,1),(pid,'Large','بڑا',750,2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_burger, 'Beef Burger', 'بیف برگر', 0, 2) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Regular','عام',600,1),(pid,'Large','بڑا',800,2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_burger, 'Chicken Burger', 'چکن برگر', 0, 3) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Regular','عام',400,1),(pid,'Large','بڑا',550,2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_burger, 'Double Patty Burger', 'ڈبل پیٹی برگر', 850, 4);

  -- Fries / snacks
  insert into public.products (category_id, name, name_ur, price, sort) values (c_fries, 'French Fries', 'فرنچ فرائز', 0, 1) returning id into pid;
  insert into public.product_variants (product_id, name, name_ur, price, sort) values (pid,'Regular','عام',200,1),(pid,'Large','بڑا',320,2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_fries, 'Masala Fries', 'مصالحہ فرائز', 250, 2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_snack, 'Chicken Nuggets (6 pcs)', 'چکن نگٹس (6)', 380, 1);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_snack, 'Hot Wings (6 pcs)', 'ہاٹ ونگز (6)', 420, 2);

  -- Drinks
  insert into public.products (category_id, name, name_ur, price, sort) values (c_drink, 'Soft Drink (345ml)', 'سافٹ ڈرنک', 150, 1);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_drink, 'Mint Margarita', 'منٹ مارگریٹا', 250, 2);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_drink, 'Mineral Water', 'منرل واٹر', 80, 3);

  -- Deals
  insert into public.products (category_id, name, name_ur, price, description, sort) values (c_deal, 'Deal 1: Burger + Fries + Drink', 'ڈیل 1', 850, 'Zinger burger, regular fries, soft drink', 1);
  insert into public.products (category_id, name, name_ur, price, description, sort) values (c_deal, 'Deal 2: Large Pizza + 2 Drinks', 'ڈیل 2', 1150, 'Any large pizza with 2 soft drinks', 2);

  -- Desserts
  insert into public.products (category_id, name, name_ur, price, sort) values (c_dessert, 'Chocolate Brownie', 'چاکلیٹ براؤنی', 300, 1);
  insert into public.products (category_id, name, name_ur, price, sort) values (c_dessert, 'Ice Cream Cup', 'آئس کریم کپ', 200, 2);

  -- Add-ons
  insert into public.addons (category_id, name, name_ur, price) values
    (c_burger, 'Extra Cheese', 'اضافی چیز', 100), (c_burger, 'Extra Chicken', 'اضافی چکن', 150), (c_burger, 'Extra Sauce', 'اضافی ساس', 50),
    (c_pizza,  'Extra Cheese', 'اضافی چیز', 150), (c_pizza,  'Extra Toppings', 'اضافی ٹاپنگ', 200), (c_pizza, 'Extra Sauce', 'اضافی ساس', 50),
    (c_fries,  'Cheese Dip', 'چیز ڈپ', 80), (c_snack, 'Extra Sauce', 'اضافی ساس', 50);
end $$;

```

## 2. Create your Super Admin (first login)

1. Open `index.html` (locally, or your live Netlify link once deployed) → click
   **"New staff — create account"** on the login screen.
2. Enter your name, a username (no spaces), and a password (6+ characters).
3. This **first account automatically becomes Super Admin** and is active immediately.
4. Every account created after that starts **inactive** with the Cashier role, until the
   Super Admin activates it and sets its real role (Manager / Cashier / Waiter / Kitchen
   Staff / Delivery Staff) from **Settings → Users** or `admin.html → Users & Roles`.
5. For a Delivery Staff account, also add a matching Rider in `admin.html → Riders &
   Incentives` and use **"Link user"** to connect it.

## 3. Upload to GitHub — no folders, just drag these 4 files

1. Create a new repository on GitHub (or open your existing one).
2. **Add file → Upload files** → drag in `index.html`, `admin.html`, `order.html`,
   `README.md` — all four at once, straight onto the page (no zip, no extracting, no folders).
3. **Commit changes**.

## 4. Deploy on Netlify (free)

1. https://app.netlify.com → **Add new site → Import an existing project** → connect the
   GitHub repo above.
2. Build command: leave **empty**. Publish directory: leave **empty** (or `.`).
3. Deploy. You'll get a `https://something.netlify.app` link.
4. Pages: `/index.html` (POS), `/admin.html` (Admin Panel), `/order.html` (customer site).

## 5. Move to youngmalang.com later

Netlify → **Domain settings → Add a custom domain** → `youngmalang.com`, then point your
domain's DNS to Netlify as it instructs. No code changes needed.

## QR ordering for tables

Each table's QR code should point to:
```
https://youngmalang.com/order.html?table=T01
```
(swap `T01` for each table's code, shown in Admin → Tables.)

## Roles at a glance

| Role | Can |
|---|---|
| **Super Admin** | everything, incl. Admin Panel, prices, users, passwords |
| **Manager** | run POS, riders, waiters, tables, cancellations, reports, most settings |
| **Cashier** | create/checkout orders, receive payments, limited discount, no prices/users |
| **Waiter** | "My Tables" — take dine-in orders, mark a ready order as Served |
| **Kitchen Staff** | Kitchen Display only — accept, prepare, mark ready |
| **Delivery Staff** | "My Deliveries" — accept/pick up/deliver their own assigned orders |

Every staff member automatically gets a permanent **Staff ID** (ST-001, ST-002…) the moment
their account is created — it shows on receipts next to their name.

## Notes

- **Real-time**: every screen updates live for every logged-in user without refreshing.
- **Language / Theme**: buttons on every screen switch English ⇄ Urdu and cycle
  Classic / Dark / Sunny themes — saved per device.
- **Receipts**: print (58mm / 80mm / A4) or download after every order/payment; NTN, Receipt
  Number, and who served the order (waiter / rider / cashier) all print automatically.
- If something ever says *"Supabase is not connected yet"*, the config values above got
  changed by mistake — check them against what's shown in section 1.
- If a page ever shows **"Could not load a required library"**, it means the visitor's
  internet connection or an ad-blocker stopped a small required script from
  cdn.jsdelivr.net from loading — ask them to check their connection and reload.
