# Campaign Hub — Roadmap

Ordering principle, in priority order:

1. **Dependency** — nothing is built before the thing it reads from exists.
2. **Risk** — the parts that can invalidate the product (gating, RLS, the two-sided
   liquidity problem) are built early, while changing them is still cheap.
3. **Demonstrability** — each slice must be showable to a real supplier or a real
   restaurant, because the commercial pitch runs in parallel with the build.

Everything ships as a reviewable slice. We stop for review after each one.

---

## MVP (v1)

### Slice 0 — Foundation *(prerequisite, not a demo)*
Vite + React + TS strict, Tailwind with design tokens as CSS variables, shadcn/ui,
Supabase project, migrations pipeline, service layer skeleton, the four screen-state
primitives (`EmptyState`, `LoadingState`, `ErrorState`, toast), the public layout with
**"Sou Restaurante" / "Sou Fornecedor"**.

*Why first:* the design tokens and the service-layer boundary are the two rules that
get violated the moment a screen is written under pressure. They have to exist before
the first screen, not after.

### Slice 1 — Auth, role routing, admin approval (F1, F2)
Signup/login, `user_roles` with a server-assigned default, protected route shells per
role, the pending/rejected account states, and the admin queue that approves suppliers
and restaurants.

*Why here:* every other table's RLS policy depends on the role helper functions and on
the `approved` status. Getting this wrong is the single most expensive mistake in the
project, and it is only cheap to fix while there are no other policies.

**Risk retired:** RLS foundations and role escalation.

### Slice 2 — Public consumer catalog (F11, F12, F13)
City filter, campaign page, participating restaurants, configurable CTA, click
tracking. Seeded with fixture data until Slices 3–4 produce real rows.

*Why before the private areas:* this is the only screen a supplier can be shown to
close a deal, and it is the surface where the "same day, same city" promise is either
felt or not. It also forces the public-read RLS and the anon column grants to be
designed early, which is the second-riskiest security surface after roles.

**Risk retired:** does the product read as a movement, or as a coupon site?

### Slice 3 — Restaurant area (F6, F8, F9, F10)
Browse, multi-select join in a single order, proof upload, Media Kit unlock, offer
editor with live preview.

*Why before the supplier area:* the restaurant side contains the gating rule — assets
unlocked only at `min_order_confirmed` — which is the core mechanic of the whole
product. It is also the harder side to acquire, so its friction needs the most
iteration time.

**Risk retired:** asset gating cannot be bypassed; the proof flow is completable by a
restaurant owner on a phone.

### Slice 4 — Supplier area (F3, F4, F5, F7, F14)
Brand profile, campaign builder, asset upload, enrollment and proof approval,
performance dashboard.

*Why after:* every supplier screen is a producer or a reviewer of data whose shape is
only settled once the restaurant side consumes it. Building the builder first
guarantees rework in the campaign schema.

**Risk retired:** the supplier can see enough performance data to justify paying again.

### Slice 5 — Admin console + Clube + email (F15, F16, F17 hardening)
City management, campaign curation, search, audit log, Clube signup and saved offers,
transactional emails, RLS test suite run end to end.

*Why last:* until Slices 1–4 exist, the admin console has nothing to curate and the
Clube has nothing to save. Admin gaps are survivable in the short term with direct
Supabase access; a broken restaurant flow is not.

---

## v2 — after the first real campaign runs

Ordered by what the first campaign will most likely prove missing.

1. **Teams per company** (A3). The moment a supplier is an agency or a marketing
   department, one login stops working. Needs a `memberships` table and a rewrite of
   every `owner_id = auth.uid()` policy into a membership lookup — which is why it is
   the first v2 item: the longer we wait, the more policies must be migrated.
2. **Waiting list and automatic capacity handling** for full campaigns.
3. **Automated NF-e validation** (A1). Removes the supplier's manual review, the
   biggest operational cost of the model, and kills duplicate-proof fraud. Depends on
   having enough real proofs to know which document types actually arrive.
4. **Supplier → restaurant broadcast** (announcements, reminders before the activation
   date) as email, not chat.
5. **Asset personalisation**: banner templates auto-composited with the restaurant's
   logo and offer copy. Highest perceived value per unit of effort, but needs a stable
   asset taxonomy from v1 usage.
6. **Restaurant dashboard**: how the restaurant's own offer performed vs the city
   average — the retention hook for the harder side of the marketplace.
7. **CSV export** for suppliers and admin.
8. **Recurring / multi-date campaigns** (A2 relaxed), only if suppliers ask.

## v3 — once liquidity exists in more than one city

1. **Chains with multiple units** (A4) — `restaurant_units`, unit-level enrollment,
   consolidated brand dashboard.
2. **Vouchers and redemption** (A5) — code generation, counter validation, redemption
   funnel. This turns "cliques" into "vendas atribuídas" and is the strongest argument
   for supplier renewal, but it requires operational discipline at the restaurant that
   only exists after several successful campaigns.
3. **Geolocation and "perto de mim"**, once a single city has enough density that a
   city filter is too coarse.
4. **Consumer app / PWA with push** for city alerts on activation days.
5. **Restaurant microsites**, gamification, in-app chat, event ticketing, payments —
   the explicit out-of-v1 list, revisited only with evidence of demand.
6. **Public API / partner integrations** (POS, delivery marketplaces).

---

## Sequencing risks we are accepting

| Risk | Where it bites | Mitigation now |
|---|---|---|
| Cold-start: no campaigns means no restaurants, no restaurants means no suppliers | Slices 2–4 demo with fixtures | Ship the public catalog early so the first supplier is sold on a live screen, and seed the first city manually with a single anchor campaign. |
| Manual proof review does not scale | Slice 4 onwards | Accepted for v1 (A1); NF-e automation is queued as v2 item 3 with the schema field (`nfe_key`) already in place. |
| One-user-per-company breaks on the first agency | v2 item 1 | All policies are written against helper functions (`current_supplier_id()`), so the migration touches the functions, not 40 policies. |
| The activation-date model is wrong for some suppliers | v2 item 8 | `activation_date` is a column, not a hardcoded assumption; adding a dates table later does not invalidate existing rows. |
| Analytics volume outgrows a plain table | v2 | `offer_events` is append-only with covering indexes and a daily rollup view from day one; partitioning is a later change to one table. |
