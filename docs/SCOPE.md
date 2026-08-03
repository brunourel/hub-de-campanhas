# Campaign Hub — Escopo v1

> **Purpose (one sentence):** Campaign Hub connects suppliers who fund ready-made
> marketing campaigns with young restaurants who lack budget and structure, so that
> every participating restaurant in a city runs the same offer on the same day and
> fills empty tables as part of a visible movement instead of an isolated promo.

**Language of these docs:** technical prose in English (mirroring the brief), all
user-facing copy in Brazilian Portuguese and quoted verbatim. Identifiers, table
names and enum values are English snake_case.

---

## 0. Assumptions

Decisions confirmed with the product owner before writing this document:

| # | Topic | Decision |
|---|---|---|
| A1 | Proof of minimum purchase | The restaurant uploads a document (NF-e PDF/XML, distributor order, or photo) and **the supplier approves or rejects it manually**. No SEFAZ/NF-e API integration in v1; the NF-e access key is stored as text for future automated validation and duplicate detection. |
| A2 | Campaign timing | Each campaign has **one national activation date** set by the supplier. Every enrolled restaurant publishes on that day. The restaurant chooses whether to join, never when to run it. |
| A3 | Accounts | **One user = one company.** The owner who signs up *is* the supplier/restaurant account. Team members with invites are v2. |
| A4 | Restaurant granularity | **One registration = one location in one city.** A chain registers each unit separately. A `restaurant_units` table is explicitly v3. |
| A5 | Consumer CTA | The CTA is **always an external link** (WhatsApp, iFood, Instagram, phone, Google Maps, own website). We track views and clicks. No vouchers, no redemption codes, no counter validation in v1. |

Assumptions made without asking (flagged so they are cheap to challenge):

| # | Topic | Assumption |
|---|---|---|
| A6 | Market & locale | Brazil only. Currency `BRL`, timezone `America/Sao_Paulo`, dates rendered `dd/MM/yyyy`, money rendered `R$ 1.234,56`. All timestamps stored as `timestamptz` in UTC. |
| A7 | Consumer auth | Browsing the public catalog **never** requires login. "Clube de Experiências" signup uses Supabase Auth (email + magic link / OTP) and only unlocks saved offers and city alerts. |
| A8 | Cities | Cities are a **curated list managed by admin**, not free text and not an IBGE import. A campaign explicitly selects the cities it runs in. |
| A9 | Email | Transactional email (approval, rejection, proof reviewed, campaign reminder) is sent from **edge functions** via a provider API key held server-side. Templates in Portuguese. Emails are v1 but the last slice built. |
| A10 | Storage | Three buckets: `campaign-assets` (private, gated), `purchase-proofs` (private), `public-media` (public: logos, covers, offer images). Gated downloads are served as short-lived signed URLs minted by an edge function, never by a direct client call. |
| A11 | Analytics identity | Consumer events are anonymous. We store a daily rotating `session_hash` (salted hash of IP + user agent + date), never a raw IP. This is our LGPD posture for v1. |
| A12 | Approval model | A restaurant needs **two green lights** to appear publicly: admin approves the restaurant account once, and the supplier approves each enrollment. |
| A13 | Enrollment capacity | A campaign may cap the number of restaurants (`max_restaurants`). When null there is no cap. Approval order is first-come, supplier-decided; there is no waiting list in v1. |
| A14 | Money | The platform handles **no payments**. Supplier billing happens offline/commercially. `min_order_amount` is a threshold to verify, never a charge. |
| A15 | Offer lifecycle | An offer auto-hides from the public catalog after its campaign activation date passes. Historical rows are kept for analytics, not deleted. |

---

## 1. Features

Each feature lists acceptance criteria as **"the user can X and sees Y"**. Every
criterion is a testable statement; the four screen states (empty / loading / error /
success) are mandatory across all data screens and are re-stated per feature only
where the behaviour is non-obvious.

### F1 — Authentication and role routing

Email + password signup and login, with a role chosen at signup (`restaurante` or
`fornecedor`); `consumidor` is created implicitly by the Clube signup; `admin` is
assigned only by another admin.

**Acceptance criteria**
- The user can sign up as a restaurant and sees the account-pending screen "Cadastro em análise", not the restaurant dashboard.
- The user can sign up as a supplier and sees the same pending state with supplier copy.
- The user can log in and is routed to the area of their role without ever seeing a flash of another role's layout.
- The user can hit a URL belonging to another role and sees a 403 page with a link back to their own area — never a blank screen and never the protected data.
- The user can reload any protected page and stays logged in and on that page.
- The user can request a password reset and sees a confirmation message whether or not the email exists (no account enumeration).
- The user cannot change their own role from the client; attempting it fails server-side and the UI never offers it.

### F2 — Admin approval of suppliers and restaurants

**Acceptance criteria**
- The admin can open "Cadastros pendentes" and sees a list of suppliers and restaurants awaiting review with submission date, city and CNPJ.
- The admin can approve an account and sees it move to "Aprovados"; the owner's next login lands on their working dashboard.
- The admin can reject an account with a mandatory reason and sees it move to "Recusados"; the owner sees the reason and an "Editar cadastro" action that returns them to `pending`.
- The admin can suspend an approved account and sees its campaigns/offers disappear from the public catalog within the same request cycle.
- With nothing pending, the admin sees the empty state "Nenhum cadastro aguardando análise", not an empty table with headers.

### F3 — Supplier brand profile

**Acceptance criteria**
- The supplier can fill in brand name, legal name, CNPJ, logo, description and contacts and sees inline validation for CNPJ (check digits) and for logo size/format before submitting.
- The supplier can save the profile and sees a success toast plus the updated logo in the header.
- The supplier can submit an invalid CNPJ that passes client validation (e.g. via devtools) and the server rejects it with a field-level error — the row is never written.
- An incomplete profile blocks campaign creation, and the supplier sees "Complete o perfil da marca para criar campanhas" with a link to the missing fields.

### F4 — Campaign builder (supplier)

A campaign carries: title, subtitle, description, the mechanic the restaurant must
run, cover image, minimum purchase (amount + human description), **one national
activation date**, enrollment window, proof deadline, optional restaurant cap,
target cities, and suggested offer copy + default CTA.

**Acceptance criteria**
- The supplier can create a campaign as `draft` and sees it listed under "Rascunhos" with an "Incompleta" badge until every required field is filled.
- The supplier can select one or more cities from the admin-curated list and sees them as removable chips; saving with zero cities is blocked with "Selecione ao menos uma cidade".
- The supplier can set an activation date in the future and sees an error if the enrollment window closes after it or if the proof deadline falls after the activation date.
- The supplier can submit the campaign for review and sees status "Em análise" with all editing locked except cancel.
- The supplier can see an approved campaign as "Publicada" with the count of restaurants enrolled and a countdown to the activation date.
- The supplier cannot edit `min_order_amount` or `activation_date` after the first enrollment is approved and sees those fields disabled with the reason "Já existe restaurante aprovado nesta campanha".

### F5 — Media Kit / Tool Kit upload and gating

**Acceptance criteria**
- The supplier can upload multiple assets (banner, post, story, print, video, PDF guideline), reorder them, and sees per-file progress plus per-file error rows when one fails while the others continue.
- The supplier can mark one asset as a public preview and sees it rendered on the campaign card; all others stay private.
- The restaurant with an enrollment below `min_order_confirmed` sees the asset grid blurred with the lock message **"Materiais liberados após a compra mínima"** and no downloadable URL exists in the DOM or network response.
- The restaurant with `min_order_confirmed` can download any asset individually or as a ZIP and sees the download start; the URL is a signed link that expires.
- A signed URL that has expired returns a friendly "Link expirado, gere novamente" state rather than an XML storage error.

### F6 — Restaurant discovery and multi-join in a single order

**Acceptance criteria**
- The restaurant can browse campaigns available for its own city and sees, per card, the brand, the activation date, the minimum purchase and how many restaurants already joined.
- The restaurant can select several campaigns and sees a running "Sua seleção" summary with the total minimum purchase across them.
- The restaurant can submit one order containing all selected campaigns and sees one confirmation listing every campaign with status "Aguardando aprovação do fornecedor".
- The restaurant can open a campaign it already joined and sees the "Participar da campanha" button replaced by its current enrollment status.
- The restaurant can reach a campaign that is full and sees "Vagas esgotadas" with the button disabled, both in the list and on the campaign page.
- With no campaigns for its city, the restaurant sees "Ainda não há campanhas para a sua cidade" plus an option to be notified — not a blank grid.
- Submitting an order where a campaign closed enrollment between page load and submit results in a partial result screen naming exactly which campaigns were accepted and which were not.

### F7 — Supplier approval of enrollments

**Acceptance criteria**
- The supplier can see pending enrollments with restaurant name, city, neighbourhood, cuisine, opening date and Instagram, and sees the campaign's remaining capacity.
- The supplier can approve an enrollment and sees it move to "Aprovadas"; the restaurant is notified and its next step becomes "Enviar comprovante".
- The supplier can reject an enrollment with a mandatory reason and sees it move to "Recusadas" with the reason visible to that restaurant only.
- The supplier can bulk-approve a selection and sees a per-row result, including which rows failed because capacity ran out mid-operation.
- The supplier can see contact details (email, WhatsApp) of restaurants **only** after approving them; before approval the contact block shows "Disponível após aprovação".

### F8 — Proof of minimum purchase

**Acceptance criteria**
- The approved restaurant can upload a proof (PDF, XML, JPG, PNG up to 10 MB), declare the amount, the purchase date, the distributor and optionally the NF-e key, and sees inline validation of the 44-digit key format.
- The restaurant can submit the proof and sees status "Comprovante em análise" plus the file name and the deadline.
- The restaurant can replace a rejected proof and sees the supplier's rejection reason above the upload field.
- The supplier can open a proof, view the file inline, and approve or reject it with a reason; approving sets the enrollment to `min_order_confirmed`.
- Approving a proof whose declared amount is below the campaign minimum requires an explicit confirmation ("Valor abaixo do mínimo. Confirmar mesmo assim?") and records who overrode it.
- The restaurant that misses the proof deadline sees the enrollment as "Prazo encerrado" and the assets stay locked.
- A proof file that fails to upload shows a retry action on that specific file and never leaves an enrollment in a half-submitted state.

### F9 — Asset unlock

**Acceptance criteria**
- The restaurant whose enrollment reaches `min_order_confirmed` can open "Meus materiais" and sees every asset of that campaign unlocked, with the activation date pinned at the top.
- The restaurant can only see assets of campaigns it has confirmed; requesting an asset from another campaign returns 403 from the edge function and the UI shows "Você não tem acesso a este material".
- The restaurant can see, for a locked campaign, exactly what is missing ("Falta enviar o comprovante" / "Comprovante em análise") instead of a generic lock.

### F10 — Offer editor (copy + CTA)

**Acceptance criteria**
- The restaurant with a confirmed enrollment can edit the offer headline, description, terms, optional image and the CTA (type + value) and sees a live preview of the public card exactly as the consumer will see it.
- The restaurant can start from the supplier's suggested copy pre-filled and sees a "Restaurar sugestão do fornecedor" action.
- The restaurant can choose CTA type WhatsApp and sees phone-mask validation; choosing iFood/site validates the URL server-side, and an invalid value is rejected even if the client is bypassed.
- The restaurant can publish the offer and sees "Publicada — vai ao ar em DD/MM"; before the activation date the offer is not visible in the public catalog.
- The restaurant can unpublish and sees the offer removed from the public catalog on the next request.
- An offer missing the CTA cannot be published and the restaurant sees which field blocks it.

### F11 — Public consumer catalog

**Acceptance criteria**
- Any visitor, logged out, can open the home page and sees the hero **"As melhores ofertas da sua cidade, no mesmo dia."** with a city selector.
- The visitor can pick a city and sees only offers active in that city, ordered by activation date then by restaurant name; the choice persists in the URL and in local storage.
- The visitor can share the URL and the recipient sees the same filtered result.
- With no live offers in the chosen city, the visitor sees a named empty state offering the Clube signup and the nearest cities with offers.
- The visitor on a slow connection sees skeleton cards, never layout shift when data arrives.
- On a failed request, the visitor sees "Não foi possível carregar as ofertas" with a retry button, not an empty catalog implying there is nothing.

### F12 — Campaign page and participating restaurants

**Acceptance criteria**
- The visitor can open a campaign page and sees the campaign story, the date, and every participating restaurant in the selected city with its individual offer.
- The visitor can click a restaurant's CTA and is taken to the external destination in a new tab, with the click recorded.
- The visitor can open a campaign whose date has passed and sees "Esta campanha já aconteceu" plus current campaigns, instead of a 404.
- The header always offers **"Sou Restaurante"** and **"Sou Fornecedor"**.

### F13 — Event tracking (views and CTA clicks)

**Acceptance criteria**
- A visitor scrolling the catalog generates one `offer_view` per offer per session, not one per scroll event.
- A visitor clicking a CTA generates one `cta_click`, and the navigation still happens if the tracking request fails.
- The tracking endpoint rejects events for offers that are not published and rate-limits by session hash; rejected events never reach the table.
- No raw IP address is ever stored, and this is verifiable by reading the table.

### F14 — Supplier performance dashboard

**Acceptance criteria**
- The supplier can open a campaign dashboard and sees restaurants enrolled, restaurants confirmed, offer views and CTA clicks, with the conversion rate between views and clicks.
- The supplier can see a per-restaurant table and sorts it by clicks or views.
- The supplier can filter by city and by date range and sees the numbers update.
- With a campaign that has not gone live, the supplier sees "Os dados aparecem após a data de ativação" instead of zeros presented as results.
- The supplier can never see another supplier's numbers, and a crafted request for another campaign's metrics returns no rows.

### F15 — Clube de Experiências (consumer)

**Acceptance criteria**
- The visitor can join the Clube with email and city and sees a confirmation; no password is required (magic link).
- The member can save an offer and sees it under "Salvos", persisted across devices.
- The member can unsubscribe in one click from any email and sees the confirmation page.
- A visitor who is not a member sees the save icon and, on click, an inline signup instead of a hard redirect that loses the page.

### F16 — Admin console

**Acceptance criteria**
- The admin can create, rename, activate and deactivate cities and sees the count of restaurants and campaigns attached before deactivating one.
- The admin can review campaigns submitted by suppliers, approve or reject with a reason, and sees the change reflected on the supplier's side.
- The admin can search restaurants and suppliers by name, CNPJ or city and sees paginated results.
- The admin can open any enrollment and see its full history (requested, decided, proof, confirmation) with timestamps and actor.
- Every approval, rejection and suspension the admin performs is written to an audit log the admin can read.

### F17 — Cross-cutting (applies to every screen)

- Every data screen implements **empty, loading, error and success** states, each with real Portuguese copy.
- All colours, radii and spacing come from CSS variables in `index.css`, mapped into Tailwind; no hardcoded values in JSX.
- All data access goes through the service layer in `src/services/`; components never call Supabase directly.
- All destructive actions require confirmation naming the object being affected.
- All forms are validated with a shared zod schema reused by the edge function for the same operation.

---

## 2. Explicitly out of v1

Listed so nobody has to guess. These are backlog, not rejections.

**From the brief:**
1. Points, badges, gamification, restaurant ranking.
2. Full restaurant microsites (own domain, menu, gallery).
3. In-app chat between supplier and restaurant.
4. Payments, invoicing, split, subscription billing.
5. Native mobile app.
6. Consumer event ticketing for the Clube.

**Added from the decisions above:**
7. Multi-user teams per company (invites, roles inside a company) — A3.
8. Chains with multiple units under one account — A4.
9. Automated NF-e validation against SEFAZ — A1.
10. Restaurant-chosen activation dates or recurring campaigns — A2.
11. Vouchers, redemption codes, counter validation, redemption reporting — A5.

**Added from scoping:**
12. Map / geolocation / "perto de mim" search — city filter only.
13. Consumer reviews, ratings or comments on offers.
14. Waiting list when a campaign is full.
15. Supplier-to-restaurant broadcast messaging or push notifications.
16. Asset personalisation (auto-generating a banner with the restaurant's logo).
17. Public API, exports beyond a simple CSV, BI integrations.
18. Internationalisation — Portuguese only, no i18n framework.
19. WhatsApp Business API integration (the CTA is a plain `wa.me` link).
20. Soft-delete/restore UI — deletions are status changes, purge is manual.

---

## 3. Definition of done for v1

The v1 is shippable when a real supplier can create a campaign, three real
restaurants in one city can join it in a single order, upload proof, get confirmed,
unlock the Media Kit, publish their offers, and a consumer with no account can open
the catalog on a phone, filter by that city, and click through to WhatsApp — with
the supplier seeing those clicks in the dashboard the next morning, and with every
RLS policy verified by a test that attempts the forbidden read and fails.
