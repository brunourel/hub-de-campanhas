# BORA — Roadmap

Ordering principle, in priority order:

1. **Dependency** — nothing is built before the thing it reads from exists.
2. **Risk** — whatever can invalidate the product (SEO rendering, the asset gate, RLS)
   is built while changing it is still cheap.
3. **Demonstrability** — every slice must be showable to a real supplier, because the
   commercial pitch runs in parallel with the build.

The consumer surface leads because discovery is the front door: it is what a supplier
is actually buying, and it is the only screen that can be sold before a single
restaurant exists.

Each slice ships and stops for review.

---

## MVP (v1)

### Slice 0 — Foundation *(prerequisite, not a demo)*

Vite + **Vike SSR** + React + TS strict, Tailwind wired to the token set in
`index.css` (blue = action, orange = identity), shadcn/ui with pill buttons and 1px
outlines, Supabase project, migrations pipeline, service layer skeleton, the four
screen-state primitives, the public shell with **"Sou Restaurante"** / **"Sou Fornecedor"**.

*Why first:* two rules break the moment a screen is written under pressure — design
tokens and the service-layer boundary. And retrofitting SSR onto a finished SPA is a
rewrite, not a refactor. Both have to exist before the first page.

**Risk retired:** the SSR + Supabase + shadcn combination actually builds and deploys.

### Slice 1 — Public consumer catalog + full SEO layer (F1–F6)

City catalog, campaign page, restaurant page, experiences, configurable CTA, click
tracking, meta/canonical/JSON-LD/sitemap/robots, OG image generation at publish time.

*Why here:* it is the primary surface and the sales asset. It also forces the anon RLS
policies and the public column grants to be designed first — the second-riskiest
security surface after roles — and it settles the URL structure while nothing is
indexed yet, which is the only moment changing it is free.

**Runs on seeded data.** There is no supplier UI yet, so the first campaigns, cities
and restaurants are inserted by a seed migration against the real schema. This is
deliberate: it proves the schema serves the public read path before any write UI is
built on top of it.

**Risk retired:** does BORA read as a movement rather than a coupon site — and does
Google actually get HTML.

### Slice 2 — Auth, role routing, admin approval (F7, F8)

Signup/login, `user_roles` with a server-assigned default, protected shells per role,
pending/rejected/suspended account states, and the admin queue.

*Why second:* every private table's policy depends on the role helpers and on
`status = 'approved'`. Getting this wrong is the most expensive mistake in the project,
and it is only cheap to fix while few policies exist.

**Risk retired:** RLS foundations and role escalation.

### Slice 3 — Restaurant area (F9–F13)

Browse, multi-select join in a single order, proof upload, Media Kit unlock, offer
editor with live preview.

*Why before the supplier area:* this side holds the gate — assets unlocked only at
`min_order_confirmed` — which is the core mechanic. It is also the harder side to
acquire, so its friction deserves the most iteration time.

**Risk retired:** the gate cannot be bypassed; a restaurant owner can complete the
proof flow on a phone.

### Slice 4 — Supplier area (F14–F18)

Brand profile, campaign builder, asset upload, enrollment and proof approval,
performance dashboard.

*Why after:* every supplier screen produces or reviews data whose shape only settles
once the restaurant side and the public pages consume it. Building the builder first
guarantees rework in the campaign schema.

**Risk retired:** the supplier sees enough performance data to justify paying again.

### Slice 5 — Admin console + email + hardening (F19, F20, F21)

City management, campaign curation, search, audit log, transactional emails, the RLS
verification matrix run end to end, Lighthouse and Rich Results checks on every public
route.

*Why last:* until Slices 1–4 exist there is nothing to curate. Admin gaps survive on
direct Supabase access for a few weeks; a broken restaurant flow does not.

---

## v2 — after the first real campaign runs

Ordered by what the first campaign will most likely prove missing.

1. **Clube de Experiências** (A7). The email list captured in v1 is the seed. Saved offers, city alerts on activation day, and exclusive events. First because it converts the traffic Slice 1 buys into an owned audience — without it, every consumer visit is spent once.
2. **Teams per company** (A3). The first agency or marketing department breaks the one-login model. Every `owner_id = auth.uid()` policy becomes a membership lookup — which is why it is early: the longer we wait, the more policies migrate.
3. **Automated NF-e validation** (A1). Removes the supplier's manual review, the biggest operational cost of the model, and kills duplicate-proof fraud. Needs real proofs first to know which documents actually arrive.
4. **Restaurant dashboard** — how its own offer performed against the city average. The retention hook for the harder side of the marketplace.
5. **Waiting list and automatic capacity handling** for full campaigns.
6. **Supplier → restaurant broadcast** (reminders before activation day) as email, not chat.
7. **Asset personalisation** — banner templates auto-composited with the restaurant's logo. Highest perceived value per unit of effort, but needs a stable asset taxonomy from v1 usage.
8. **Editorial/content layer** — city guides and campaign recaps, to earn the long-tail queries the catalog pages cannot rank for on their own.
9. **CSV export** for suppliers and admin.
10. **Recurring or multi-date campaigns** (A2 relaxed), only if suppliers ask.

## v3 — once liquidity exists in more than one city

1. **Chains with multiple units** (A4) — `restaurant_units`, unit-level enrollment, consolidated brand dashboard.
2. **Vouchers and redemption** (A5) — turns "cliques" into "vendas atribuídas", the strongest argument for supplier renewal. Requires operational discipline at the restaurant that only exists after several successful campaigns.
3. **Experiences as a first-class entity** (B2) — venue, capacity, RSVP, and eventually ticketing.
4. **Geolocation and "perto de mim"**, once a single city is dense enough that a city filter is too coarse.
5. **Consumer PWA with push** for activation-day alerts.
6. **Points, gamification, microsites, in-app chat, payments** — the explicit out-of-v1 list, revisited only with evidence.
7. **Public API and partner integrations** (POS, delivery marketplaces).

---

## Sequencing risks we are accepting

| Risk | Where it bites | Mitigation now |
|---|---|---|
| Cold start: a discovery product with nothing to discover | Slice 1 ships before any supplier exists | Seeded data against the real schema, and one anchor campaign in one city sold manually before launch. The catalog is the sales asset, so it must exist before the first supplier, not after. |
| SEO with thin content damages the domain | Slice 1 | Restaurant pages only become public with their first offer (B4); everything else is `noindex` **and** absent from the sitemap. We would rather index 20 real pages than 400 empty ones. |
| SSR complexity slows every subsequent slice | Slices 2–5 | SSR is confined to public routes; private areas stay client-rendered in the same build. The boundary is a routing convention, decided in Slice 0. |
| Manual proof review does not scale | Slice 4 onwards | Accepted for v1 (A1); `nfe_key` is already stored and unique, so v2 automation is additive. |
| One user per company breaks on the first agency | v2 item 2 | Policies are written against helper functions (`current_supplier_id()`), so the migration touches five functions, not forty policies. |
| The single-national-date model is wrong for some suppliers | v2 item 10 | `activation_date` is a column, not a hardcoded assumption; adding a dates table later does not invalidate existing rows. |
| Analytics volume outgrows a plain table | v2 | `offer_events` is append-only with covering indexes and a daily rollup view from day one; partitioning is a later change to one table. |
| URLs change after indexing | any slice | `slug_redirects` exists from Slice 1 (A15) — an indexed URL never 404s. |
