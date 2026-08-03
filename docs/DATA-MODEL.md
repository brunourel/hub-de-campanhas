# Campaign Hub — Data Model

Postgres on Supabase. **RLS is enabled on every table in `public`, without exception**,
including tables that look harmless (`cities`, `offer_events`). A table with RLS
enabled and no policy denies everything, which is the correct default.

Conventions:

- `id uuid primary key default gen_random_uuid()` unless stated otherwise.
- `created_at timestamptz not null default now()`, `updated_at timestamptz not null default now()` maintained by a `set_updated_at()` trigger.
- Money is `numeric(12,2)`, never float.
- Foreign keys are `on delete restrict` by default; cascades are called out explicitly.
- Storage paths are stored as text keys, never as full URLs.

The four roles referenced in every policy table below:

| Role | Meaning in policies |
|---|---|
| **anon** | Not logged in — the consumer browsing the public catalog. |
| **consumer** | Logged in, Clube member. Same public read as `anon`, plus their own club rows. |
| **restaurant** | Owner of exactly one `restaurants` row (A3, A4). |
| **supplier** | Owner of exactly one `suppliers` row (A3). |
| **admin** | Full read; writes limited to curation and approval columns. |

---

## 1. Enums

```sql
create type app_role          as enum ('admin','supplier','restaurant','consumer');
create type approval_status   as enum ('pending','approved','rejected','suspended');
create type campaign_status   as enum ('draft','pending_review','approved','rejected','published','closed','archived');
create type order_status      as enum ('draft','submitted','partially_approved','approved','rejected','cancelled');
create type enrollment_status as enum ('pending_approval','approved','proof_submitted','min_order_confirmed','rejected','cancelled','expired');
create type proof_status      as enum ('submitted','under_review','approved','rejected');
create type proof_type        as enum ('nfe_pdf','nfe_xml','order_photo','distributor_receipt','other');
create type asset_kind        as enum ('banner','social_post','story','print','video','guideline_pdf','logo','other');
create type offer_status      as enum ('draft','published','unpublished');
create type cta_type          as enum ('whatsapp','ifood','instagram','phone','maps','website');
create type offer_event_type  as enum ('offer_view','cta_click','campaign_view','offer_save');
create type club_status       as enum ('active','unsubscribed');
```

**The gate.** `enrollment_status = 'min_order_confirmed'` is the single condition that
unlocks `campaign_assets`. It is enforced in three independent places: the RLS policy
on `campaign_assets`, the storage policy on the `campaign-assets` bucket, and the edge
function that mints signed URLs. Any one of them failing still leaves the assets
locked.

---

## 2. Helper functions

All are `security definer`, `stable`, `set search_path = public`, and owned by the
migration role. They exist so that policies never self-reference the table they
protect (which is how RLS recursion bugs happen) and so the A3 → v2 migration to
multi-user teams touches five functions instead of forty policies.

```sql
create or replace function public.has_role(_user_id uuid, _role app_role)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.user_roles ur
                 where ur.user_id = _user_id and ur.role = _role);
$$;

create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select public.has_role(auth.uid(), 'admin');
$$;

-- returns the supplier id only when the account is approved
create or replace function public.current_supplier_id()
returns uuid language sql stable security definer set search_path = public as $$
  select s.id from public.suppliers s
  where s.owner_id = auth.uid() and s.status = 'approved' limit 1;
$$;

create or replace function public.current_restaurant_id()
returns uuid language sql stable security definer set search_path = public as $$
  select r.id from public.restaurants r
  where r.owner_id = auth.uid() and r.status = 'approved' limit 1;
$$;

-- THE GATE
create or replace function public.has_unlocked_campaign(_campaign_id uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1 from public.campaign_enrollments e
    where e.campaign_id = _campaign_id
      and e.restaurant_id = public.current_restaurant_id()
      and e.status = 'min_order_confirmed'
  );
$$;

-- an offer the public is allowed to see
create or replace function public.is_offer_public(_offer_id uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1
    from public.restaurant_offers o
    join public.campaign_enrollments e on e.id = o.enrollment_id
    join public.campaigns c            on c.id = o.campaign_id
    join public.restaurants r          on r.id = o.restaurant_id
    where o.id = _offer_id
      and o.status = 'published'
      and e.status = 'min_order_confirmed'
      and c.status = 'published'
      and r.status = 'approved'
      and c.activation_date >= (now() at time zone 'America/Sao_Paulo')::date
  );
$$;
```

> **Column-level protection.** RLS filters rows, not columns. Sensitive columns
> (`cnpj`, `legal_name`, `owner_id`, `contact_email`, `contact_phone`) are protected by
> `revoke all on <table> from anon, authenticated;` followed by an explicit
> `grant select (col, col, ...)` listing only public columns. The grant lists are given
> per table below and are part of the migration, not an afterthought.

---

## 3. Tables

### 3.1 `user_roles`

Roles live in their own table, never on `profiles`, so that a compromised or careless
profile update can never grant a role.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null | → `auth.users(id)` on delete cascade |
| role | app_role not null | |
| granted_by | uuid null | → `auth.users(id)`, null for signup default |
| created_at | timestamptz | |

Indexes: `unique (user_id, role)`, `index (role)`.

A `handle_new_user()` trigger on `auth.users` inserts `profiles` + the signup role
(`restaurant`, `supplier` or `consumer`, read from `raw_user_meta_data`). `admin` is
never assignable this way — the trigger rejects it.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ | ✗ | ✗ | ✗ |
| consumer / restaurant / supplier | own rows (`user_id = auth.uid()`) | ✗ | ✗ | ✗ |
| admin | all | all except `role = 'admin'` on self | ✗ (revoke + insert instead) | all |

Writes by non-admins happen only through the signup trigger (definer) or an edge
function using the service role.

### 3.2 `profiles`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | = `auth.users(id)`, on delete cascade |
| full_name | text not null | |
| phone | text null | E.164 |
| avatar_path | text null | `public-media` |
| created_at / updated_at | timestamptz | |

Indexes: pk only.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ | ✗ | ✗ | ✗ |
| consumer / restaurant / supplier | own row | own row (trigger) | own row, cannot change `id` | ✗ |
| admin | all | ✗ | ✗ | ✗ |

Deliberately **not** readable across users: a supplier sees restaurant contacts through
`restaurants`, never through `profiles`.

### 3.3 `cities`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| name | text not null | |
| state_uf | char(2) not null | |
| slug | text not null | url-safe, e.g. `sao-paulo-sp` |
| is_active | boolean not null default true | |
| created_at | timestamptz | |

Indexes: `unique (slug)`, `unique (lower(name), state_uf)`, `index (is_active) where is_active`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_active = true` | ✗ | ✗ | ✗ |
| restaurant / supplier | `is_active = true` | ✗ | ✗ | ✗ |
| admin | all | all | all | ✗ (deactivate instead; FK restrict would block anyway) |

Grants: `grant select (id, name, state_uf, slug) on cities to anon, authenticated;`

### 3.4 `suppliers`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| owner_id | uuid not null | → `auth.users(id)`, **unique** (A3) |
| brand_name | text not null | |
| legal_name | text not null | |
| cnpj | text not null | digits only, check-digit validated server-side |
| slug | text not null | |
| logo_path | text null | `public-media` |
| description | text null | |
| website_url / contact_email / contact_phone | text null | |
| status | approval_status not null default 'pending' | |
| rejection_reason | text null | |
| reviewed_by | uuid null / reviewed_at | timestamptz null |
| created_at / updated_at | timestamptz | |

Indexes: `unique (owner_id)`, `unique (cnpj)`, `unique (slug)`, `index (status)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `status = 'approved'` (public columns only) | ✗ | ✗ | ✗ |
| restaurant | `status = 'approved'` (public columns only) | ✗ | ✗ | ✗ |
| supplier | own row (`owner_id = auth.uid()`), all columns | own row once, forced `status='pending'` | own row, **only** when `status in ('pending','rejected','approved')`; cannot write `status`, `reviewed_by`, `reviewed_at`, `rejection_reason` (blocked by trigger) | ✗ |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` only | ✗ |

Grants: `grant select (id, brand_name, slug, logo_path, description, website_url) on suppliers to anon, authenticated;` — `cnpj`, `legal_name`, `owner_id` and contacts are **not** granted.

### 3.5 `restaurants`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| owner_id | uuid not null | → `auth.users(id)`, **unique** (A3) |
| name | text not null | |
| slug | text not null | |
| legal_name | text not null / cnpj text not null | |
| city_id | uuid not null | → `cities(id)` (A4: exactly one) |
| address_line / neighborhood / postal_code | text | |
| cuisine_type | text null | |
| opened_at | date null | drives the "menos de 1 ano" segment |
| seats | int null | |
| logo_path / cover_path | text null | `public-media` |
| instagram_handle | text null | |
| whatsapp_phone | text null | E.164 |
| contact_email | text null | |
| status | approval_status not null default 'pending' | |
| rejection_reason / reviewed_by / reviewed_at | | |
| created_at / updated_at | timestamptz | |

Indexes: `unique (owner_id)`, `unique (cnpj)`, `unique (slug)`, `index (city_id)`,
`index (status)`, `index (city_id, status) where status = 'approved'`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `status = 'approved'` **and** the restaurant has at least one public offer (public columns only) | ✗ | ✗ | ✗ |
| restaurant | own row, all columns | own row once, forced `status='pending'` | own row; cannot write `status`, `reviewed_*`, `rejection_reason` | ✗ |
| supplier | rows that have an enrollment in one of the supplier's campaigns — **contact columns only after that enrollment is `approved` or beyond** (F7); enforced by two policies + a restricted view `supplier_restaurant_contacts` | ✗ | ✗ | ✗ |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` | ✗ |

Grants (anon/authenticated): `id, name, slug, city_id, neighborhood, cuisine_type, logo_path, cover_path, instagram_handle`. Not granted: `cnpj`, `legal_name`, `owner_id`, `address_line`, `postal_code`, `contact_email`, `whatsapp_phone`.
The supplier's access to contact data goes through the view, which applies the
`enrollment.status <> 'pending_approval'` condition.

### 3.6 `campaigns`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| supplier_id | uuid not null | → `suppliers(id)` |
| slug | text not null | |
| title / subtitle / description | text | |
| mechanics_description | text not null | what the restaurant must offer |
| cover_path | text null | `public-media` |
| min_order_amount | numeric(12,2) not null check (> 0) | |
| min_order_description | text not null | e.g. "10 barris de 30L" |
| activation_date | date not null | **A2 — single national date** |
| enrollment_opens_at / enrollment_closes_at | timestamptz not null | |
| proof_deadline | date not null | |
| max_restaurants | int null check (> 0) | null = uncapped (A13) |
| default_offer_headline / default_offer_description / default_offer_terms | text null | suggested copy |
| default_cta_type | cta_type null / default_cta_value text null | |
| status | campaign_status not null default 'draft' | |
| rejection_reason / reviewed_by / reviewed_at | | |
| published_at | timestamptz null | |
| created_at / updated_at | timestamptz | |

Checks: `enrollment_closes_at::date <= activation_date`, `proof_deadline <= activation_date`, `enrollment_opens_at < enrollment_closes_at`.

Indexes: `unique (slug)`, `index (supplier_id)`, `index (status)`,
`index (activation_date)`, `index (status, activation_date) where status = 'published'`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `status = 'published'` and `activation_date >= today` (public columns) | ✗ | ✗ | ✗ |
| restaurant | `status = 'published'` and the campaign targets the restaurant's city (via `campaign_cities`), **plus** any campaign it is enrolled in regardless of date | ✗ | ✗ | ✗ |
| supplier | own campaigns (`supplier_id = current_supplier_id()`), any status | own campaigns, forced `status='draft'` | own campaigns while `status in ('draft','rejected')`; cannot write `status` directly (state transitions go through an edge function); `min_order_amount` and `activation_date` frozen once an approved enrollment exists (trigger) | own campaigns while `status = 'draft'` and no enrollments |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` | ✗ |

Grants (anon/authenticated): everything except `reviewed_by`, `rejection_reason`.

### 3.7 `campaign_cities`

Join table — which cities a campaign runs in.

| Column | Type |
|---|---|
| campaign_id | uuid not null → `campaigns(id)` on delete cascade |
| city_id | uuid not null → `cities(id)` |
| pk (campaign_id, city_id) |

Index: `index (city_id)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant | rows whose campaign is publicly visible | ✗ | ✗ | ✗ |
| supplier | rows of own campaigns | own campaigns while `status in ('draft','rejected')` | ✗ | same condition as insert |
| admin | all | ✗ | ✗ | ✗ |

### 3.8 `campaign_assets` — **gated**

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| campaign_id | uuid not null | → `campaigns(id)` on delete cascade |
| kind | asset_kind not null | |
| title | text not null / description text null | |
| storage_path | text not null | `campaign-assets/campaigns/{campaign_id}/{id}.{ext}` — **private bucket** |
| file_name / mime_type | text not null | |
| size_bytes | bigint not null / width int null / height int null | |
| is_public_preview | boolean not null default false | one watermarked teaser may be public |
| sort_order | int not null default 0 | |
| created_at | timestamptz | |

Indexes: `index (campaign_id)`, `index (campaign_id, sort_order)`,
`index (campaign_id) where is_public_preview`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_public_preview = true` **and** campaign publicly visible | ✗ | ✗ | ✗ |
| restaurant | `is_public_preview = true` for visible campaigns, **plus all assets where `has_unlocked_campaign(campaign_id)`** | ✗ | ✗ | ✗ |
| supplier | assets of own campaigns | own campaigns | own campaigns | own campaigns, blocked once campaign is `published` |
| admin | all | ✗ | ✗ | ✗ |

A row being selectable never yields a file: `storage_path` is a bucket key, and the
bucket has its own policy (§5). The client asks the `get-asset-url` edge function,
which re-checks `has_unlocked_campaign` with the caller's JWT before minting a
60-second signed URL.

### 3.9 `campaign_orders`

The "join several campaigns in a single order" container (F6).

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| restaurant_id | uuid not null | → `restaurants(id)` |
| status | order_status not null default 'draft' | derived from its enrollments |
| total_min_amount | numeric(12,2) not null default 0 | snapshot at submit |
| submitted_at | timestamptz null | |
| created_at / updated_at | timestamptz | |

Indexes: `index (restaurant_id)`, `index (restaurant_id, status)`,
`unique (restaurant_id) where status = 'draft'` — one open cart per restaurant.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | own orders | own, forced `status='draft'` | own while `status = 'draft'` (submit goes through an edge function) | own while `status = 'draft'` |
| supplier | ✗ — suppliers see enrollments, never another supplier's basket | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

### 3.10 `campaign_enrollments`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| campaign_id | uuid not null | → `campaigns(id)` |
| restaurant_id | uuid not null | → `restaurants(id)` |
| order_id | uuid null | → `campaign_orders(id)` |
| status | enrollment_status not null default 'pending_approval' | |
| requested_at | timestamptz not null default now() | |
| decided_at | timestamptz null / decided_by uuid null | supplier decision |
| rejection_reason | text null | |
| confirmed_amount | numeric(12,2) null | value validated from the proof |
| confirmed_at | timestamptz null / confirmed_by uuid null | |
| override_below_minimum | boolean not null default false | set when a supplier confirms below `min_order_amount` (F8) |
| notes | text null | supplier-private note |
| created_at / updated_at | timestamptz | |

Indexes: `unique (campaign_id, restaurant_id)`, `index (campaign_id, status)`,
`index (restaurant_id, status)`, `index (order_id)`,
`index (status) where status = 'approved'` (deadline sweeper).

State machine (all transitions performed by edge functions, never by a raw client update):

```
pending_approval ──approve──▶ approved ──proof sent──▶ proof_submitted
       │                          │                          │
       │                          │◀────── proof rejected ────┤
       │                          │                          ▼
       └──reject──▶ rejected      └──deadline──▶ expired   min_order_confirmed  ← THE GATE
                                                                  │
restaurant may cancel while pending_approval/approved ──▶ cancelled
```

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | own (`restaurant_id = current_restaurant_id()`), excluding `notes` | own, only for a `published` campaign inside its enrollment window, targeting its city, forced `status='pending_approval'` | own, **only** `status → 'cancelled'` while `status in ('pending_approval','approved')` | ✗ |
| supplier | enrollments of own campaigns | ✗ | own campaigns' rows: `status`, `decided_*`, `rejection_reason`, `confirmed_*`, `override_below_minimum`, `notes` | ✗ |
| admin | all | ✗ | `status` (unblock/correct), always audited | ✗ |

`notes` is revoked from the restaurant at column level.

### 3.11 `purchase_proofs`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| enrollment_id | uuid not null | → `campaign_enrollments(id)` on delete cascade |
| submitted_by | uuid not null | → `auth.users(id)` |
| storage_path | text not null | `purchase-proofs/{restaurant_id}/{enrollment_id}/{id}.{ext}` — private |
| file_name / mime_type | text not null | |
| size_bytes | bigint not null check (<= 10485760) | |
| document_type | proof_type not null | |
| nfe_key | text null check (nfe_key ~ '^\d{44}$') | stored for v2 automation and duplicate detection |
| declared_amount | numeric(12,2) not null check (> 0) | |
| purchase_date | date not null | |
| distributor_name | text null | |
| status | proof_status not null default 'submitted' | |
| reviewed_by / reviewed_at / rejection_reason | | |
| created_at | timestamptz | |

Indexes: `index (enrollment_id)`, `index (status)`,
`unique (nfe_key) where nfe_key is not null` — the same invoice cannot confirm two enrollments.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | proofs of own enrollments | own enrollments, only while enrollment is `approved` or the previous proof was `rejected`, and only before `proof_deadline`; forced `status='submitted'` | ✗ | ✗ |
| supplier | proofs of enrollments in own campaigns | ✗ | `status`, `reviewed_*`, `rejection_reason` on own campaigns' proofs | ✗ |
| admin | all | ✗ | `status`, `reviewed_*` | ✗ |

The declared amount is never trusted: the edge function that approves a proof
re-reads `campaigns.min_order_amount` server-side and demands the explicit override
flag when the value is lower.

### 3.12 `restaurant_offers`

The restaurant's own public offer for one confirmed enrollment (F10). One offer per
enrollment. `campaign_id`, `restaurant_id` and `city_id` are denormalised (kept in
sync by trigger) so the public catalog query filters by city without joining four
tables under an RLS policy.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| enrollment_id | uuid not null unique | → `campaign_enrollments(id)` on delete cascade |
| campaign_id / restaurant_id / city_id | uuid not null | denormalised, trigger-maintained |
| headline | text not null check (length ≤ 80) | |
| description | text not null check (length ≤ 400) | |
| terms | text null | |
| image_path | text null | `public-media`; falls back to campaign cover |
| cta_type | cta_type not null | |
| cta_value | text not null | phone or URL, normalised + validated server-side |
| status | offer_status not null default 'draft' | |
| published_at | timestamptz null | |
| created_at / updated_at | timestamptz | |

Indexes: `unique (enrollment_id)`, `index (city_id, status) where status = 'published'`,
`index (campaign_id)`, `index (restaurant_id)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_offer_public(id)` | ✗ | ✗ | ✗ |
| restaurant | own offers, any status | own, **only when the enrollment is `min_order_confirmed`** and belongs to it | own; publishing requires a valid `cta_value` (trigger + edge function) | own while `status = 'draft'` |
| supplier | offers attached to own campaigns, any status | ✗ | ✗ | ✗ |
| admin | all | ✗ | `status` (takedown) | ✗ |

### 3.13 `offer_events`

Append-only analytics. **No client ever writes here.** The public site calls the
`track-offer-event` edge function, which validates that the offer is public,
rate-limits by `session_hash`, and inserts with the service role.

| Column | Type | Notes |
|---|---|---|
| id | bigint generated always as identity, pk | high volume |
| offer_id | uuid null | → `restaurant_offers(id)` on delete set null |
| campaign_id / restaurant_id / city_id | uuid not null | denormalised for aggregation |
| event_type | offer_event_type not null | |
| occurred_at | timestamptz not null default now() | |
| session_hash | text not null | salted hash of IP + UA + date (A11) — **no raw IP, ever** |
| referrer_host | text null / user_agent_family text null / device_kind text null | |
| club_member_id | uuid null | → `club_members(id)`, only for logged-in members |

Indexes: `index (offer_id, occurred_at desc)`,
`index (campaign_id, event_type, occurred_at desc)`,
`index (city_id, occurred_at desc)`,
`unique (offer_id, session_hash, event_type, (occurred_at::date))` for `offer_view` — one view per offer per session per day (F13).

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ (write-only surface, and even the write is indirect) | ✗ | ✗ | ✗ |
| restaurant | events of own offers | ✗ | ✗ | ✗ |
| supplier | events of own campaigns | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

A `offer_daily_metrics` view aggregates by `(offer_id, date, event_type)` and inherits
the same visibility through `security_invoker = true`.

### 3.14 `club_members`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null unique | → `auth.users(id)` on delete cascade (A7) |
| email | text not null | snapshot for mailing |
| full_name | text null / phone text null | |
| city_id | uuid not null | → `cities(id)` |
| accepts_marketing | boolean not null default true | |
| accepted_terms_at | timestamptz not null | LGPD consent timestamp |
| status | club_status not null default 'active' | |
| unsubscribe_token | uuid not null default gen_random_uuid() | one-click unsubscribe |
| created_at / updated_at | timestamptz | |

Indexes: `unique (user_id)`, `index (city_id) where status = 'active'`, `unique (unsubscribe_token)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ (signup goes through an edge function) | ✗ | ✗ | ✗ |
| consumer | own row | own row once | own row, cannot write `unsubscribe_token` | own row (LGPD deletion) |
| restaurant / supplier | ✗ — **the member list is never sold or exposed** | ✗ | ✗ | ✗ |
| admin | aggregate counts via a view; the raw table only for support | ✗ | `status` | own-request deletion |

### 3.15 `club_saved_offers`

| Column | Type |
|---|---|
| club_member_id | uuid not null → `club_members(id)` on delete cascade |
| offer_id | uuid not null → `restaurant_offers(id)` on delete cascade |
| created_at | timestamptz |
| pk (club_member_id, offer_id) |

Index: `index (offer_id)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ | ✗ | ✗ | ✗ |
| consumer | own rows | own rows, only when `is_offer_public(offer_id)` | ✗ | own rows |
| restaurant / supplier | ✗ (they see the aggregate `offer_save` count via `offer_events`) | ✗ | ✗ | ✗ |
| admin | ✗ | ✗ | ✗ | ✗ |

### 3.16 `audit_log`

| Column | Type | Notes |
|---|---|---|
| id | bigint identity pk | |
| actor_id | uuid null | → `auth.users(id)` |
| actor_role | app_role null | |
| action | text not null | `supplier.approved`, `enrollment.confirmed`, … |
| entity_table | text not null / entity_id uuid not null | |
| before / after | jsonb null | changed columns only |
| reason | text null | |
| created_at | timestamptz | |

Indexes: `index (entity_table, entity_id, created_at desc)`, `index (actor_id, created_at desc)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant / supplier | ✗ | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

Written only by definer triggers and edge functions.

---

## 4. Relations at a glance

```
auth.users ─1:1─ profiles
auth.users ─1:N─ user_roles
auth.users ─1:1─ suppliers            (A3)
auth.users ─1:1─ restaurants          (A3)
auth.users ─1:1─ club_members         (A7)

cities ─1:N─ restaurants
cities ─N:N─ campaigns                (campaign_cities)
cities ─1:N─ club_members

suppliers ─1:N─ campaigns
campaigns ─1:N─ campaign_assets                       [gated]
campaigns ─1:N─ campaign_enrollments ─N:1─ restaurants
campaign_orders ─1:N─ campaign_enrollments            (multi-join in one order)
campaign_enrollments ─1:N─ purchase_proofs
campaign_enrollments ─1:1─ restaurant_offers
restaurant_offers ─1:N─ offer_events
restaurant_offers ─N:N─ club_members  (club_saved_offers)
```

---

## 5. Storage buckets and policies

| Bucket | Public | Contents | Path convention |
|---|---|---|---|
| `public-media` | yes | logos, covers, offer images | `suppliers/{id}/…`, `restaurants/{id}/…`, `offers/{id}/…` |
| `campaign-assets` | **no** | Media Kit / Tool Kit | `campaigns/{campaign_id}/{asset_id}.{ext}` |
| `purchase-proofs` | **no** | NF-e, receipts, photos | `{restaurant_id}/{enrollment_id}/{proof_id}.{ext}` |

`storage.objects` policies:

- `public-media` — read: everyone. Write/update/delete: the owner of the entity in the first two path segments (`suppliers/{current_supplier_id()}/…`, etc.).
- `campaign-assets` — read: `is_admin()`, the owning supplier, **or** a restaurant with `has_unlocked_campaign(<campaign_id parsed from the path>)`. Write: owning supplier only. In practice the client never reads the bucket directly; the `get-asset-url` edge function mints a 60-second signed URL after re-checking the gate. The bucket policy is the second lock, not the only one.
- `purchase-proofs` — read: the owning restaurant, the supplier of that enrollment's campaign, `is_admin()`. Write: the owning restaurant only, and only under its own `{restaurant_id}/` prefix. No public read under any condition.

## 6. Edge functions (server-side validation, service role never in the frontend)

| Function | Why it cannot be a client write |
|---|---|
| `submit-campaign-order` | validates the whole cart atomically: campaign published, city matches, window open, capacity available, no duplicate enrollment; returns per-campaign results (F6). |
| `decide-enrollment` | supplier approve/reject with capacity re-check under a lock. |
| `submit-proof` | file type/size, NF-e key format and uniqueness, deadline check. |
| `review-proof` | compares `declared_amount` against `min_order_amount`, requires the override flag, flips the enrollment to `min_order_confirmed`, writes the audit row. |
| `get-asset-url` | re-checks the gate, mints a short-lived signed URL. |
| `publish-offer` | normalises and validates `cta_value` per `cta_type`, checks the enrollment is confirmed. |
| `track-offer-event` | validates offer visibility, rate-limits, computes `session_hash`, inserts without exposing the table. |
| `join-club` / `unsubscribe-club` | LGPD consent record, token handling. |
| `admin-review-account` / `admin-review-campaign` | approval + audit + email. |
| `expire-enrollments` (cron) | flips `approved` past `proof_deadline` to `expired`. |

## 7. RLS verification matrix

Every policy is proven by a test that **attempts the forbidden read and asserts zero
rows or a 403**, not just by a test that the allowed read works.

| # | Actor | Attempt | Expected |
|---|---|---|---|
| 1 | anon | `select * from campaign_assets` | only `is_public_preview` rows |
| 2 | restaurant with enrollment `approved` (no proof) | select assets of that campaign | 0 rows |
| 3 | restaurant with `proof_submitted` | select assets of that campaign | 0 rows |
| 4 | restaurant with `min_order_confirmed` | select assets of that campaign | all rows |
| 5 | restaurant A confirmed in campaign X | select assets of campaign Y | 0 rows |
| 6 | restaurant A | download signed URL of restaurant B's proof | 403 |
| 7 | supplier A | select `campaigns` of supplier B | 0 rows |
| 8 | supplier A | select `offer_events` of supplier B's campaigns | 0 rows |
| 9 | supplier | select `restaurants.whatsapp_phone` of a `pending_approval` enrollment | null / 0 rows |
| 10 | restaurant | `update campaign_enrollments set status='min_order_confirmed'` on own row | rejected |
| 11 | restaurant | `update restaurants set status='approved'` on own row | rejected |
| 12 | any authenticated user | `insert into user_roles (auth.uid(),'admin')` | rejected |
| 13 | any authenticated user | `insert into offer_events …` | rejected |
| 14 | supplier | select `club_members` | 0 rows |
| 15 | anon | select an offer whose enrollment is not confirmed | 0 rows |
| 16 | anon | select an offer whose campaign date has passed | 0 rows |
| 17 | anon | select `restaurants.cnpj` | permission denied for column |
| 18 | restaurant | select `campaign_enrollments.notes` of own row | permission denied for column |
| 19 | supplier | `update campaigns set min_order_amount` after an approved enrollment | rejected by trigger |
| 20 | restaurant | insert a second proof with an `nfe_key` already used | unique violation |
