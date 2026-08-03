# BORA — Data Model

Postgres on Supabase. **RLS is enabled on every table in `public`, without exception**,
including tables that look harmless (`cities`, `offer_events`, `slug_redirects`). A
table with RLS enabled and no policy denies everything, which is the correct default.

> **SSR does not bypass RLS.** The Vike server (B1) queries Supabase with the **anon
> key**, exactly like a browser. Server rendering changes where the HTML is built, never
> who is allowed to read. The service role key exists only inside edge functions and is
> never present in any bundle, server or client.

Conventions:

- `id uuid primary key default gen_random_uuid()` unless stated otherwise.
- `created_at timestamptz not null default now()`, `updated_at` maintained by a `set_updated_at()` trigger.
- Money is `numeric(12,2)`, never float.
- Foreign keys are `on delete restrict` unless a cascade is stated.
- Storage paths are stored as bucket keys, never as full URLs.

Roles referenced in every policy table:

| Role | Meaning in policies |
|---|---|
| **anon** | Not logged in — the consumer, and the SSR server rendering on their behalf. |
| **consumer** | Logged in. In v1 identical to `anon` in what it may read (A7). |
| **restaurant** | Owner of exactly one `restaurants` row (A3, A4). |
| **supplier** | Owner of exactly one `suppliers` row (A3). |
| **admin** | Full read; writes limited to curation and approval columns. |

---

## 1. Enums

```sql
create type app_role          as enum ('admin','supplier','restaurant','consumer');
create type approval_status   as enum ('pending','approved','rejected','suspended');
create type campaign_kind     as enum ('oferta','experiencia');                 -- B2
create type campaign_status   as enum ('draft','pending_review','approved','rejected','published','closed','archived');
create type order_status      as enum ('draft','submitted','partially_approved','approved','rejected','cancelled');
create type enrollment_status as enum ('pending_approval','approved','proof_submitted','min_order_confirmed','rejected','cancelled','expired');
create type proof_status      as enum ('submitted','under_review','approved','rejected');
create type proof_type        as enum ('nfe_pdf','nfe_xml','order_photo','distributor_receipt','other');
create type asset_kind        as enum ('banner','social_post','story','print','video','guideline_pdf','logo','other');
create type offer_status      as enum ('draft','published','unpublished');
create type cta_type          as enum ('whatsapp','ifood','instagram','phone','maps','website');
create type offer_event_type  as enum ('city_view','campaign_view','restaurant_view','offer_view','cta_click','offer_save');
create type entity_kind       as enum ('campaign','restaurant','city');         -- slug_redirects
create type club_status       as enum ('active','unsubscribed');
```

**The gate.** `enrollment_status = 'min_order_confirmed'` is the single condition that
unlocks `campaign_assets`. It is enforced in three independent places: the RLS policy
on `campaign_assets`, the storage policy on the `campaign-assets` bucket, and the edge
function that mints signed URLs. Any one of them failing still leaves the assets locked.

---

## 2. Helper functions

`security definer`, `stable`, `set search_path = public`. They exist so policies never
self-reference the table they protect (the classic RLS recursion bug) and so the v2
migration to multi-user teams touches five functions instead of forty policies.

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

-- Visible to the public at all: powers /restaurante/:slug and /campanha/:slug history.
-- Deliberately has NO date condition — an indexed URL must never 404 (A15, A16).
create or replace function public.is_offer_visible(_offer_id uuid)
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
      and c.status in ('published','closed')
      and r.status = 'approved'
  );
$$;
```

**Visible ≠ live.** `is_offer_visible` decides *readability*. Whether an offer appears
in the city catalog, carries the **"Campanha ativa"** badge, or is rendered as history
is a query filter on `campaigns.activation_date`, not a permission:

| Concept | Condition | Used by |
|---|---|---|
| live | `activation_date = hoje` | catalog, "Campanha ativa" badge |
| upcoming | `activation_date > hoje` | catalog, "Acontece em DD/MM" |
| history | `activation_date < hoje` | restaurant page, campaign page, `noindex` |

> **Column-level protection.** RLS filters rows, not columns. `cnpj`, `legal_name`,
> `owner_id`, `contact_email`, `whatsapp_phone` and `address_line` are protected by
> `revoke all on <table> from anon, authenticated;` then an explicit
> `grant select (col, …)` listing only the public columns. The grant lists below are
> part of the migration, not an afterthought.

---

## 3. Tables

### 3.1 `user_roles`

Roles live in their own table, never on `profiles`, so a careless profile update can
never grant a role.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null | → `auth.users(id)` on delete cascade |
| role | app_role not null | |
| granted_by | uuid null | → `auth.users(id)`; null for the signup default |
| created_at | timestamptz | |

Indexes: `unique (user_id, role)`, `index (role)`.

A `handle_new_user()` trigger on `auth.users` inserts `profiles` plus the signup role
read from `raw_user_meta_data`. `admin` is never assignable this way — the trigger
rejects it.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ | ✗ | ✗ | ✗ |
| consumer / restaurant / supplier | own rows | ✗ | ✗ | ✗ |
| admin | all | all, except granting `admin` to self | ✗ | all |

### 3.2 `profiles`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | = `auth.users(id)` on delete cascade |
| full_name | text not null | |
| phone | text null | E.164 |
| avatar_path | text null | `public-media` |
| created_at / updated_at | timestamptz | |

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ | ✗ | ✗ | ✗ |
| consumer / restaurant / supplier | own row | own row (trigger) | own row, cannot change `id` | ✗ |
| admin | all | ✗ | ✗ | ✗ |

Deliberately not readable across users: a supplier reaches restaurant contacts through
`restaurants`, never through `profiles`.

### 3.3 `cities`

Every active city is an indexable landing page, so it carries its own SEO fields.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| name | text not null / state_uf char(2) not null | |
| slug | text not null | `sao-paulo-sp` — the `:cidade` segment |
| seo_title | text null / seo_description text null | fall back to generated copy |
| hero_image_path | text null | `public-media` |
| is_active | boolean not null default true | |
| created_at / updated_at | timestamptz | |

Indexes: `unique (slug)`, `unique (lower(name), state_uf)`, `index (is_active) where is_active`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_active = true` | ✗ | ✗ | ✗ |
| restaurant / supplier | `is_active = true` | ✗ | ✗ | ✗ |
| admin | all | all | all | ✗ (deactivate; FK restrict would block anyway) |

Grants (anon/authenticated): `id, name, state_uf, slug, seo_title, seo_description, hero_image_path`.

### 3.4 `suppliers`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| owner_id | uuid not null | → `auth.users(id)`, **unique** (A3) |
| brand_name | text not null / legal_name text not null | |
| cnpj | text not null | digits only, check digits validated server-side |
| slug | text not null | |
| logo_path | text null | `public-media` |
| description | text null | |
| website_url / contact_email / contact_phone | text null | |
| status | approval_status not null default 'pending' | |
| rejection_reason | text null / reviewed_by uuid null / reviewed_at timestamptz null | |
| created_at / updated_at | timestamptz | |

Indexes: `unique (owner_id)`, `unique (cnpj)`, `unique (slug)`, `index (status)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant | `status = 'approved'`, public columns only | ✗ | ✗ | ✗ |
| supplier | own row, all columns | own row once, forced `status='pending'` | own row; cannot write `status`, `reviewed_*`, `rejection_reason` (trigger) | ✗ |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` only | ✗ |

Grants (anon/authenticated): `id, brand_name, slug, logo_path, description, website_url`. **Not** granted: `cnpj`, `legal_name`, `owner_id`, contacts.

### 3.5 `restaurants`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| owner_id | uuid not null | → `auth.users(id)`, **unique** (A3) |
| name | text not null / slug text not null | slug = `nome-cidade` on collision |
| legal_name | text not null / cnpj text not null | |
| city_id | uuid not null | → `cities(id)` — exactly one (A4) |
| address_line / neighborhood / postal_code | text | `address_line` is private |
| latitude / longitude | numeric null | for `Restaurant` JSON-LD, not for search (v3) |
| cuisine_type | text null / price_range text null | `servesCuisine`, `priceRange` |
| opening_hours | jsonb null | `openingHoursSpecification` |
| opened_at | date null | drives the "menos de 1 ano" segment |
| seats | int null | |
| logo_path / cover_path / og_image_path | text null | `public-media` |
| instagram_handle | text null | |
| whatsapp_phone / contact_email | text null | private |
| seo_description | text null | |
| **first_offer_published_at** | timestamptz null | **B4 — set on the first published offer, never cleared** |
| **published_offers_count** | int not null default 0 | trigger-maintained; drives the catalog |
| status | approval_status not null default 'pending' | |
| rejection_reason / reviewed_by / reviewed_at | | |
| created_at / updated_at | timestamptz | |

Indexes: `unique (owner_id)`, `unique (cnpj)`, `unique (slug)`, `index (city_id)`,
`index (status)`, `index (city_id, status) where status = 'approved'`,
`index (first_offer_published_at) where first_offer_published_at is not null` — the sitemap query.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `status = 'approved'` **and** `first_offer_published_at is not null` (B4), public columns only | ✗ | ✗ | ✗ |
| restaurant | own row, all columns | own row once, forced `status='pending'` | own row; cannot write `status`, `reviewed_*`, `first_offer_published_at`, `published_offers_count` | ✗ |
| supplier | restaurants enrolled in own campaigns; **contact columns only once that enrollment is `approved` or beyond**, through the `supplier_restaurant_contacts` view | ✗ | ✗ | ✗ |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` | ✗ |

Grants (anon/authenticated): `id, name, slug, city_id, neighborhood, cuisine_type, price_range, opening_hours, latitude, longitude, logo_path, cover_path, og_image_path, instagram_handle, seo_description, opened_at`.
**Not** granted: `cnpj`, `legal_name`, `owner_id`, `address_line`, `postal_code`, `contact_email`, `whatsapp_phone`, `seats`.

> An approved restaurant with no published offer is invisible to `anon` — the 404 and
> the sitemap exclusion in F3 are enforced by the policy, not by the router.

### 3.6 `campaigns`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| supplier_id | uuid not null | → `suppliers(id)` |
| **kind** | campaign_kind not null default 'oferta' | **B2 — `experiencia` drives `Event` JSON-LD and `/experiencias/:cidade`** |
| slug | text not null | the `:slug` segment |
| title / subtitle / description | text | |
| mechanics_description | text not null | what the restaurant must run |
| cover_path | text null | `public-media` |
| **og_image_path** | text null | **B3 — composed at publish time** |
| og_image_generated_at | timestamptz null | regenerate when title/cover change |
| seo_title / seo_description | text null | fall back to generated copy |
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

Indexes: `unique (slug)`, `index (supplier_id)`, `index (status)`, `index (activation_date)`,
`index (status, kind, activation_date) where status = 'published'` — the catalog and `/experiencias`,
`index (status, published_at) where status in ('published','closed')` — the sitemap.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `status in ('published','closed')` — past campaigns stay readable so their URLs never 404 (A16) | ✗ | ✗ | ✗ |
| restaurant | same as anon, **plus** any campaign it is enrolled in, any status | ✗ | ✗ | ✗ |
| supplier | own campaigns, any status | own, forced `status='draft'` | own while `status in ('draft','rejected')`; `status` transitions only via edge function; `min_order_amount` and `activation_date` frozen once an approved enrollment exists (trigger) | own while `draft` and no enrollments |
| admin | all | ✗ | `status`, `rejection_reason`, `reviewed_*` | ✗ |

Grants (anon/authenticated): everything except `reviewed_by`, `rejection_reason`, `default_*`.

### 3.7 `campaign_cities`

| Column | Type |
|---|---|
| campaign_id | uuid not null → `campaigns(id)` on delete cascade |
| city_id | uuid not null → `cities(id)` |
| pk (campaign_id, city_id) |

Index: `index (city_id)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant | rows whose campaign is publicly readable | ✗ | ✗ | ✗ |
| supplier | own campaigns | own while `status in ('draft','rejected')` | ✗ | same condition as insert |
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
`unique (campaign_id) where is_public_preview` — the teaser is exclusive.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_public_preview = true` and campaign publicly readable | ✗ | ✗ | ✗ |
| restaurant | the teaser, **plus every asset where `has_unlocked_campaign(campaign_id)`** | ✗ | ✗ | ✗ |
| supplier | own campaigns | own campaigns | own campaigns | own campaigns, blocked once `published` |
| admin | all | ✗ | ✗ | ✗ |

Reading the row never yields a file: `storage_path` is a bucket key and the bucket has
its own policy (§5). The client calls `get-asset-url`, which re-checks the gate with
the caller's JWT before minting a 60-second signed URL.

### 3.9 `campaign_orders`

The "join several campaigns in one order" container (F10).

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| restaurant_id | uuid not null | → `restaurants(id)` |
| status | order_status not null default 'draft' | derived from its enrollments |
| total_min_amount | numeric(12,2) not null default 0 | snapshot at submit |
| submitted_at | timestamptz null | |
| created_at / updated_at | timestamptz | |

Indexes: `index (restaurant_id, status)`, `unique (restaurant_id) where status = 'draft'` — one open cart per restaurant.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | own | own, forced `draft` | own while `draft` (submit goes through an edge function) | own while `draft` |
| supplier | ✗ — suppliers see enrollments, never another supplier's basket | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

### 3.10 `campaign_enrollments`

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| campaign_id | uuid not null → `campaigns(id)` / restaurant_id uuid not null → `restaurants(id)` | |
| order_id | uuid null | → `campaign_orders(id)` |
| status | enrollment_status not null default 'pending_approval' | |
| requested_at | timestamptz not null default now() | |
| decided_at | timestamptz null / decided_by uuid null / rejection_reason text null | supplier decision |
| confirmed_amount | numeric(12,2) null / confirmed_at timestamptz null / confirmed_by uuid null | |
| override_below_minimum | boolean not null default false | set when confirmed below `min_order_amount` (F17) |
| notes | text null | supplier-private |
| created_at / updated_at | timestamptz | |

Indexes: `unique (campaign_id, restaurant_id)`, `index (campaign_id, status)`,
`index (restaurant_id, status)`, `index (order_id)`, `index (status) where status = 'approved'` (deadline sweeper).

State machine — every transition runs in an edge function, never a raw client update:

```
pending_approval ──approve──▶ approved ──proof sent──▶ proof_submitted
       │                         │                          │
       │                         │◀───── proof rejected ─────┤
       │                         │                          ▼
       └──reject──▶ rejected     └──deadline──▶ expired   min_order_confirmed  ← THE GATE
                                                                  │
restaurant may cancel while pending_approval / approved ──▶ cancelled
```

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | own, `notes` revoked at column level | own, only for a `published` campaign inside its enrollment window targeting its city, forced `pending_approval` | own, **only** `status → 'cancelled'` while `pending_approval` or `approved` | ✗ |
| supplier | enrollments of own campaigns | ✗ | own campaigns' rows: `status`, `decided_*`, `rejection_reason`, `confirmed_*`, `override_below_minimum`, `notes` | ✗ |
| admin | all | ✗ | `status` (unblock/correct), always audited | ✗ |

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
| nfe_key | text null check (nfe_key ~ '^\d{44}$') | stored for v2 automation |
| declared_amount | numeric(12,2) not null check (> 0) | |
| purchase_date | date not null / distributor_name text null | |
| status | proof_status not null default 'submitted' | |
| reviewed_by / reviewed_at / rejection_reason | | |
| created_at | timestamptz | |

Indexes: `index (enrollment_id)`, `index (status)`,
`unique (nfe_key) where nfe_key is not null` — the same invoice cannot confirm two enrollments.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ | ✗ | ✗ | ✗ |
| restaurant | proofs of own enrollments | own enrollments, only while `approved` or after a rejected proof, and only before `proof_deadline`; forced `status='submitted'` | ✗ | ✗ |
| supplier | proofs of enrollments in own campaigns | ✗ | `status`, `reviewed_*`, `rejection_reason` on own campaigns' proofs | ✗ |
| admin | all | ✗ | `status`, `reviewed_*` | ✗ |

`declared_amount` is never trusted: `review-proof` re-reads `campaigns.min_order_amount`
server-side and demands the explicit override flag when the value is lower.

### 3.12 `restaurant_offers`

The restaurant's public offer for one confirmed enrollment (F13). One per enrollment.
`campaign_id`, `restaurant_id` and `city_id` are denormalised (trigger-maintained) so
the SSR catalog query filters by city without joining four tables under RLS.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| enrollment_id | uuid not null unique | → `campaign_enrollments(id)` on delete cascade |
| campaign_id / restaurant_id / city_id | uuid not null | denormalised |
| headline | text not null check (length ≤ 80) | |
| description | text not null check (length ≤ 400) | |
| terms | text null | |
| image_path | text null | `public-media`; falls back to the campaign cover |
| cta_type | cta_type not null / cta_value text not null | normalised + validated server-side |
| status | offer_status not null default 'draft' | |
| published_at | timestamptz null | first publication; also sets `restaurants.first_offer_published_at` |
| created_at / updated_at | timestamptz | |

Indexes: `unique (enrollment_id)`, `index (city_id, status) where status = 'published'` — the catalog,
`index (campaign_id, status)`, `index (restaurant_id, status)`.

A trigger on publish sets `restaurants.first_offer_published_at` (once, never cleared)
and maintains `published_offers_count` — the two columns behind B4.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | `is_offer_visible(id)` — no date condition (A16) | ✗ | ✗ | ✗ |
| restaurant | own offers, any status | own, **only when the enrollment is `min_order_confirmed`** | own; publishing requires a valid `cta_value` (trigger + edge function) | own while `draft` |
| supplier | offers attached to own campaigns, any status | ✗ | ✗ | ✗ |
| admin | all | ✗ | `status` (takedown) | ✗ |

### 3.13 `slug_redirects`

A15 — an indexed URL never 404s. Written by a trigger whenever a slug changes.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| entity | entity_kind not null | campaign / restaurant / city |
| old_slug | text not null | |
| new_slug | text not null | |
| created_at | timestamptz | |

Indexes: `unique (entity, old_slug)`, `index (entity, new_slug)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant / supplier | all (it is public routing data) | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

The SSR router resolves a 404 against this table before rendering the error page and
answers 301 when it finds a match. Chains are collapsed to the final slug on write.

### 3.14 `offer_events`

Append-only analytics. **No client ever writes here.** The public pages call the
`track-offer-event` edge function, which validates visibility, rate-limits by
`session_hash`, and inserts with the service role.

| Column | Type | Notes |
|---|---|---|
| id | bigint generated always as identity, pk | |
| event_type | offer_event_type not null | includes `city_view`, `campaign_view`, `restaurant_view` for the SEO funnel |
| offer_id | uuid null | → `restaurant_offers(id)` on delete set null |
| campaign_id / restaurant_id | uuid null | denormalised |
| city_id | uuid not null | every public page belongs to a city |
| occurred_at | timestamptz not null default now() | |
| session_hash | text not null | salted hash of IP + UA + date (A11) — **no raw IP, ever** |
| referrer_host | text null / utm_source text null / device_kind text null | `referrer_host` only, never the full URL |
| created_date | date generated always as (…) stored | dedup + rollup key |

Indexes: `index (offer_id, occurred_at desc)`, `index (campaign_id, event_type, occurred_at desc)`,
`index (city_id, occurred_at desc)`,
`unique (offer_id, session_hash, event_type, created_date) where event_type = 'offer_view'` — one view per offer per session per day (F5).

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer | ✗ — write-only surface, and even the write is indirect | ✗ | ✗ | ✗ |
| restaurant | events of own offers | ✗ | ✗ | ✗ |
| supplier | events of own campaigns | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

`offer_daily_metrics` aggregates by `(offer_id, created_date, event_type)` with
`security_invoker = true`, inheriting the same visibility.

### 3.15 `club_members` — **v2 table, v1 uses it only as an email capture (A7)**

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid null unique | → `auth.users(id)` on delete cascade — **null in v1**, filled when the Clube ships |
| email | text not null | |
| full_name | text null / phone text null | |
| city_id | uuid not null | → `cities(id)` — what they asked to be alerted about |
| source | text not null default 'city_empty_state' | where the capture happened |
| accepts_marketing | boolean not null default true | |
| accepted_terms_at | timestamptz not null | LGPD consent timestamp |
| status | club_status not null default 'active' | |
| unsubscribe_token | uuid not null default gen_random_uuid() | one-click unsubscribe |
| created_at / updated_at | timestamptz | |

Indexes: `unique (user_id) where user_id is not null`, `unique (lower(email), city_id)`,
`index (city_id) where status = 'active'`, `unique (unsubscribe_token)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon | ✗ — signup goes through the `join-club` edge function, so the table is never exposed to a form | ✗ | ✗ | ✗ |
| consumer | own row (v2) | ✗ | own row, cannot write `unsubscribe_token` | own row (LGPD deletion) |
| restaurant / supplier | ✗ — **the list is never exposed or sold** | ✗ | ✗ | ✗ |
| admin | aggregate counts via a view; raw rows only for support | ✗ | `status` | on request |

`club_saved_offers` (member ↔ offer, pk on both columns) is defined in v2 with the
Clube itself; it has no purpose while there is no login.

### 3.16 `audit_log`

| Column | Type | Notes |
|---|---|---|
| id | bigint identity pk | |
| actor_id | uuid null / actor_role app_role null | |
| action | text not null | `supplier.approved`, `enrollment.confirmed`, `campaign.taken_down`, … |
| entity_table | text not null / entity_id uuid not null | |
| before / after | jsonb null | changed columns only |
| reason | text null | |
| created_at | timestamptz | |

Indexes: `index (entity_table, entity_id, created_at desc)`, `index (actor_id, created_at desc)`.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| anon / consumer / restaurant / supplier | ✗ | ✗ | ✗ | ✗ |
| admin | all | ✗ | ✗ | ✗ |

Written only by definer triggers and edge functions. No edit or delete path exists for
anyone, including admins (F19).

---

## 4. Relations at a glance

```
auth.users ─1:1─ profiles
auth.users ─1:N─ user_roles
auth.users ─1:1─ suppliers            (A3)
auth.users ─1:1─ restaurants          (A3)

cities ─1:N─ restaurants
cities ─N:N─ campaigns                (campaign_cities)
cities ─1:N─ club_members

suppliers ─1:N─ campaigns  (kind: oferta | experiencia)
campaigns ─1:N─ campaign_assets                       [gated]
campaigns ─1:N─ campaign_enrollments ─N:1─ restaurants
campaign_orders ─1:N─ campaign_enrollments            (multi-join in one order)
campaign_enrollments ─1:N─ purchase_proofs
campaign_enrollments ─1:1─ restaurant_offers
restaurant_offers ─1:N─ offer_events

slug_redirects ─▶ campaigns | restaurants | cities    (301, by slug)
```

## 5. Public routes → queries

| Route | Reads | Indexable when |
|---|---|---|
| `/` | active cities | always |
| `/ofertas/:cidade` | offers where `is_offer_visible` and `activation_date >= hoje`, joined to campaign + restaurant | city active |
| `/experiencias/:cidade` | same, filtered `campaigns.kind = 'experiencia'` | city active |
| `/campanha/:slug` | campaign `published`/`closed` + its visible offers, city-filtered | `status = 'published'` and `activation_date >= hoje`; otherwise `noindex`, still 200 |
| `/restaurante/:slug` | restaurant with `first_offer_published_at is not null` + its visible offers | always once public (B4) |
| `/sitemap.xml` | active cities + published campaigns + indexable restaurants | — |

## 6. Storage buckets and policies

| Bucket | Public | Contents | Path convention |
|---|---|---|---|
| `public-media` | yes | logos, covers, offer images, **OG images** | `suppliers/{id}/…`, `restaurants/{id}/…`, `offers/{id}/…`, `og/campaigns/{id}.png` |
| `campaign-assets` | **no** | Media Kit / Tool Kit | `campaigns/{campaign_id}/{asset_id}.{ext}` |
| `purchase-proofs` | **no** | NF-e, receipts, photos | `{restaurant_id}/{enrollment_id}/{proof_id}.{ext}` |

- `public-media` — read: everyone. Write: the owner of the entity in the first two path segments. `og/` is writable only by the service role (B3).
- `campaign-assets` — read: `is_admin()`, the owning supplier, or a restaurant with `has_unlocked_campaign(<campaign_id from the path>)`. Write: owning supplier. In practice the client never reads the bucket directly; `get-asset-url` mints a 60-second signed URL after re-checking the gate. The bucket policy is the second lock, not the only one.
- `purchase-proofs` — read: the owning restaurant, the supplier of that enrollment's campaign, `is_admin()`. Write: the owning restaurant, only under its own `{restaurant_id}/` prefix. No public read under any condition.

## 7. Edge functions

The service role key lives here and nowhere else.

| Function | Why it cannot be a client write |
|---|---|
| `submit-campaign-order` | validates the whole cart atomically — published, city matches, window open, capacity free, no duplicate — and returns per-campaign results (F10). |
| `decide-enrollment` | supplier approve/reject with a capacity re-check under lock. |
| `submit-proof` | file type/size, NF-e key format and uniqueness, deadline. |
| `review-proof` | compares `declared_amount` to `min_order_amount`, requires the override flag, flips the enrollment to `min_order_confirmed`, writes the audit row. |
| `get-asset-url` | re-checks the gate, mints a short-lived signed URL. |
| `publish-offer` | normalises and validates `cta_value` per `cta_type`, confirms the enrollment, sets `first_offer_published_at`. |
| `generate-og-image` | B3 — composes the share card once at publish and stores it in `public-media/og/`. |
| `track-offer-event` | validates visibility, rate-limits, computes `session_hash`, inserts without exposing the table. |
| `join-club` | A7 — email capture with LGPD consent, so `club_members` needs no anon insert policy. |
| `admin-review-account` / `admin-review-campaign` | approval + audit + email. |
| `expire-enrollments` (cron) | flips `approved` past `proof_deadline` to `expired`. |
| `refresh-sitemap` (cron + on publish) | regenerates `sitemap.xml` from the DB. |

## 8. RLS verification matrix

Every policy is proven by a test that **attempts the forbidden read and asserts zero
rows or a 403** — not merely that the allowed read works.

| # | Actor | Attempt | Expected |
|---|---|---|---|
| 1 | anon | `select * from campaign_assets` | only the `is_public_preview` row |
| 2 | restaurant with enrollment `approved` (no proof) | select that campaign's assets | 0 rows |
| 3 | restaurant with `proof_submitted` | select that campaign's assets | 0 rows |
| 4 | restaurant with `min_order_confirmed` | select that campaign's assets | all rows |
| 5 | restaurant A confirmed in campaign X | select assets of campaign Y | 0 rows |
| 6 | restaurant A | fetch a signed URL for restaurant B's proof | 403 |
| 7 | supplier A | select supplier B's campaigns | 0 rows |
| 8 | supplier A | select `offer_events` of supplier B's campaigns | 0 rows |
| 9 | supplier | select `whatsapp_phone` of a `pending_approval` enrollment's restaurant | 0 rows / denied |
| 10 | restaurant | `update campaign_enrollments set status='min_order_confirmed'` on its own row | rejected |
| 11 | restaurant | `update restaurants set status='approved'` on its own row | rejected |
| 12 | any authenticated user | `insert into user_roles (auth.uid(),'admin')` | rejected |
| 13 | any authenticated user | `insert into offer_events …` | rejected |
| 14 | supplier | `select * from club_members` | 0 rows |
| 15 | anon | select an offer whose enrollment is not confirmed | 0 rows |
| 16 | anon | select an approved restaurant with `first_offer_published_at is null` | 0 rows |
| 17 | anon | `select cnpj from restaurants` | permission denied for column |
| 18 | restaurant | `select notes from campaign_enrollments` on its own row | permission denied for column |
| 19 | supplier | `update campaigns set min_order_amount` after an approved enrollment | rejected by trigger |
| 20 | restaurant | insert a second proof reusing an existing `nfe_key` | unique violation |
| 21 | anon | select a `draft` or `pending_review` campaign | 0 rows |
| 22 | anon | request `/sitemap.xml` | contains only active cities, published campaigns and indexable restaurants |
| 23 | anon (SSR server) | any public page query with the anon key | returns exactly what a browser would get — SSR grants no extra visibility |
