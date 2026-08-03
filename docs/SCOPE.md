# BORA — Escopo v1

**BORA** — Benefícios, Ofertas, Restaurantes, Ativações.
*"Bora é onde o consumidor acha o rolê e a marca acha o restaurante."*

> **Purpose (one sentence):** BORA is a public discovery product where anyone searching
> for offers and things to do in their city finds restaurants running the same
> supplier-funded campaign on the same day — and that consumer demand is what makes a
> supplier pay to put its brand, its budget and its ready-made Media Kit behind
> restaurants that could never afford marketing alone.

**Language of these docs:** technical prose in English, all user-facing copy in
Brazilian Portuguese and quoted verbatim. Identifiers, tables and enums are English
snake_case.

---

## 0. Assumptions

### 0.1 Decisions confirmed with the product owner

| # | Topic | Decision |
|---|---|---|
| A1 | Proof of minimum purchase | The restaurant uploads a document (NF-e PDF/XML, distributor order, or photo) and **the supplier approves or rejects it manually**. No ERP or SEFAZ integration in the MVP. The NF-e access key is stored as text for future automation and duplicate detection. |
| A2 | Campaign timing | Each campaign has **one national activation date**. Every enrolled restaurant publishes on that day. The restaurant chooses whether to join, never when to run it. |
| A3 | Accounts | **One user = one company.** The owner who signs up *is* the supplier/restaurant account. Teams with invites are v2. |
| A4 | Restaurant granularity | **One registration = one location in one city.** Chains register each unit separately. `restaurant_units` is v3. |
| A5 | Consumer CTA | Always an **external link** (WhatsApp, iFood, Instagram, phone, Google Maps, own site). Views and clicks are tracked. No vouchers, no redemption codes in v1. |
| B1 | Rendering | **Vike (SSR on top of Vite).** Public pages are server-rendered on every request: real HTML, per-page meta, canonical and JSON-LD, fresh data. The private areas stay client-rendered in the same codebase and the same build. |
| B2 | Experiences | An experience is **a kind of campaign**, not a separate entity: `campaigns.kind ∈ ('oferta','experiencia')`. `Event` JSON-LD is generated from the campaign's national activation date. `/experiencias/:cidade` is the catalog filtered by kind, with its own canonical. |
| B3 | OG images | Composed **once at publish time** by an edge function and stored in `public-media`. Served by CDN — no per-request rendering, no crawler timeouts. |
| B4 | Restaurant page indexing | `/restaurante/:slug` becomes public and enters the sitemap when the restaurant publishes its **first offer**, and stays online forever after that (history + upcoming dates). Approved restaurants with no offer yet are `noindex` and absent from the sitemap. |

### 0.2 Assumptions made without asking

| # | Topic | Assumption |
|---|---|---|
| A6 | Market & locale | Brazil only. `BRL`, `America/Sao_Paulo`, dates `dd/MM/yyyy`, money `R$ 1.234,56`. All timestamps stored `timestamptz` in UTC. `pt-BR` is the only locale; no i18n framework. |
| A7 | Clube de Experiências | **Out of v1** (confirmed). v1 ships only an email capture — "Avise-me quando houver ofertas em {cidade}" — writing to `club_members` with no login and no emails sent yet. Saved offers, city alerts and exclusive events are v2. If even the capture is unwanted, it is one form and one edge function to remove. |
| A8 | Cities | A **curated list managed by admin**, not free text and not an IBGE import. Each campaign explicitly selects its cities. A city page only exists once the city is active. |
| A9 | Email | Transactional email (approvals, rejections, proof reviewed, activation-day reminder) is sent from edge functions with the provider key held server-side. Portuguese templates. Built in the last slice. |
| A10 | Storage | Three buckets: `campaign-assets` (private, gated), `purchase-proofs` (private), `public-media` (public: logos, covers, offer and OG images). Gated downloads are short-lived signed URLs minted by an edge function, never a direct client read. |
| A11 | Analytics identity | Consumer events are anonymous: a daily rotating `session_hash` (salted hash of IP + user agent + date). **No raw IP is ever stored.** This is the LGPD posture for v1. |
| A12 | Approval model | A restaurant needs **two green lights** to appear publicly: admin approves the account once, the supplier approves each enrollment. |
| A13 | Capacity | A campaign may cap restaurants (`max_restaurants`); null means uncapped. First-come, supplier-decided. No waiting list in v1. |
| A14 | Money | The platform processes **no payments**. Supplier billing is commercial and offline. `min_order_amount` is a threshold to verify, never a charge. |
| A15 | URL permanence | Slugs are permanent. Renaming an entity keeps the old slug alive as a 301 (`slug_redirects`). An indexed URL never 404s and never silently changes meaning. |
| A16 | Past content | A finished campaign page stays online as history with `noindex` removed only if it still carries value; it never 404s and it always links to what is live now. Offer rows are kept for analytics, never deleted. |
| A17 | Fonts | Google Sans is licensed-only. The stack ships `Google Sans Flex` → `Roboto` → `system-ui` and swaps in Google Sans behind a single CSS variable if a licence arrives. |
| A18 | Brand orange | `#EA5B0C` is a placeholder in `--brand-600`. Swapping it is a one-line change in `index.css` — no JSX touches it. |

---

## 1. Features

Acceptance criteria are written as **"the user can X and sees Y"**. Empty, loading,
error and success states are mandatory on every data screen (F20) and are only
re-stated per feature where the behaviour is non-obvious.

---

### Block A — Consumer discovery (the front door, built first)

### F1 — Public catalog by city

Route `/ofertas/:cidade`. Server-rendered. No login, ever.

**Acceptance criteria**
- Any visitor can open the home page and sees the hero **"As melhores ofertas da sua cidade, no mesmo dia."** with a city selector and the header CTAs **"Sou Restaurante"** / **"Sou Fornecedor"**.
- The visitor can pick a city and lands on `/ofertas/:cidade`, seeing only offers live in that city, ordered by activation date then restaurant name.
- The visitor can view the page with JavaScript disabled and still sees the full list of offers in the HTML source.
- The visitor can share the URL and the recipient sees exactly the same page.
- The visitor sees a **"Campanha ativa"** badge on offers running today, and "Acontece em DD/MM" on upcoming ones.
- With no live offers in that city, the visitor sees a named empty state, the "Avise-me" capture, and the nearest cities that do have offers — never a blank grid.
- With an unknown or deactivated city in the URL, the visitor sees the city selector with "Não encontramos essa cidade." and the response is a proper 404 status.
- On a failed data load, the visitor sees "Não foi possível carregar as ofertas." with a retry — never an empty list implying there is nothing.

### F2 — Campaign page

Route `/campanha/:slug`. The campaign's story, its date, and every participating
restaurant in the visitor's selected city.

**Acceptance criteria**
- The visitor can open a campaign page and sees the brand, the campaign story, the activation date and the participating restaurants with their individual offers.
- The visitor can switch city on the page and sees the participating restaurants change without losing the campaign context.
- The visitor can open a campaign whose date has passed and sees "Esta campanha já aconteceu." plus what is live now — status 200, never a 404, because the link is already shared.
- The visitor can open a campaign that has no restaurant in the selected city and sees "Nenhum restaurante confirmado em {cidade} ainda." with a city switcher.
- A visitor hitting an old slug after a rename is 301-redirected to the current URL.
- An unknown slug returns a real 404 page in Portuguese with links to the live catalog.

### F3 — Restaurant page

Route `/restaurante/:slug`. Exists publicly from the restaurant's first published
offer onwards (B4).

**Acceptance criteria**
- The visitor can open a restaurant page and sees its name, neighbourhood, city, cuisine, cover, Instagram, its live offers and its past campaigns.
- The visitor can reach the page from any offer card in the catalog.
- The visitor can open the page of a restaurant with no live offer and sees "Sem ofertas ativas no momento." plus its history and the city catalog — status 200.
- An approved restaurant that has never published an offer has no public page: the URL returns 404 and the slug is absent from the sitemap.

### F4 — Experiences

Route `/experiencias/:cidade`, the same catalog filtered to `kind = 'experiencia'` (B2).

**Acceptance criteria**
- The visitor can open `/experiencias/:cidade` and sees only experience-type campaigns in that city, with date and participating restaurants.
- The visitor can see each experience carry `Event` structured data with its date, city and participating venues.
- With no experiences in the city, the visitor sees "Ainda não há experiências em {cidade}." plus the offers catalog for the same city.
- The page declares its own canonical and never duplicates `/ofertas/:cidade` in the index.

### F5 — Configurable CTA and click tracking

**Acceptance criteria**
- The visitor can click an offer's CTA and is taken to the external destination in a new tab.
- The click is recorded as `cta_click` before navigating, and **the navigation still happens if tracking fails**.
- A visitor scrolling the catalog generates one `offer_view` per offer per session per day, not one per scroll event.
- The tracking endpoint rejects events for offers that are not public and rate-limits by session hash; rejected events never reach the table.
- No raw IP address is ever stored, and this is verifiable by reading the table.
- An offer with a malformed destination hides the CTA and shows "Contato indisponível no momento." — never a broken link.

### F6 — SEO and share layer

**Acceptance criteria**
- A crawler can request `/ofertas/:cidade`, `/campanha/:slug`, `/restaurante/:slug` and `/experiencias/:cidade` and receives complete HTML with content, without executing JavaScript.
- Every public page carries a unique `<title>`, a unique meta description built from its own data, and a self-referencing `<link rel="canonical">`.
- A campaign page carries `Offer` JSON-LD, a restaurant page carries `Restaurant` JSON-LD, and an experience page carries `Event` JSON-LD — each validating in Google's Rich Results Test.
- The developer can open `/sitemap.xml` and sees every active city, every published campaign and every indexable restaurant, with `lastmod` reflecting real changes.
- The developer can open `/robots.txt` and sees the sitemap declared, the private areas (`/app`, `/admin`, `/fornecedor`, `/restaurante-area`) disallowed.
- A page that must not be indexed (approved restaurant with no offer, draft, unpublished offer) emits `noindex` **and** is absent from the sitemap.
- Pasting any public URL into WhatsApp shows a card with the campaign's own OG image, title and description — generated at publish time (B3), never a generic placeholder.
- A renamed slug keeps the old URL working via 301, verifiable by requesting it.

---

### Block B — Access and identity

### F7 — Authentication and role routing

**Acceptance criteria**
- The user can sign up as restaurant or supplier and sees "Cadastro em análise", not a working dashboard.
- The user can log in and lands in the area of their role, with no flash of another role's layout.
- The user can hit a URL belonging to another role and sees a 403 page with a link back to their own area — never a blank screen, never the protected data.
- The user can reload any private page and stays logged in on that page.
- The user can request a password reset and sees the same confirmation whether or not the email exists (no account enumeration).
- The user cannot change their own role from the client; the attempt fails server-side and the UI never offers it.
- A logged-in user browsing the public catalog sees it exactly as an anonymous visitor does, plus a link to their area.

### F8 — Admin approval of suppliers and restaurants

**Acceptance criteria**
- The admin can open "Cadastros pendentes" and sees suppliers and restaurants awaiting review with submission date, city and CNPJ.
- The admin can approve an account and sees it move to "Aprovados"; the owner's next login lands on their working dashboard.
- The admin can reject with a mandatory reason and sees it move to "Recusados"; the owner sees the reason verbatim and an "Editar cadastro" action that returns the account to `pending`.
- The admin can suspend an approved account and sees its offers leave the public catalog on the next request.
- With nothing pending, the admin sees "Nenhum cadastro aguardando análise." — not an empty table with headers.

---

### Block C — Restaurant

### F9 — Discover campaigns

**Acceptance criteria**
- The restaurant can browse campaigns targeting its own city and sees, per card, the brand, the activation date, the minimum purchase and how many restaurants already joined.
- The restaurant can open a campaign and sees **"Participar da campanha"**, replaced by its current status once enrolled.
- The restaurant can see a full campaign marked "Vagas esgotadas" with the action disabled, and a closed one marked "Inscrições encerradas".
- With no campaigns for its city, it sees "Ainda não há campanhas para a sua cidade." plus a notify action.
- With everything already joined, it sees "Você já participa de todas as campanhas disponíveis." and a link to its campaigns.

### F10 — Multi-join in a single order

**Acceptance criteria**
- The restaurant can select several campaigns and sees a running "Sua seleção" summary with each minimum purchase and the total.
- The restaurant can submit one order covering the whole selection and sees a confirmation listing every campaign as "Aguardando aprovação do fornecedor".
- When a campaign fills up between page load and submit, the restaurant sees a **partial result** naming exactly which campaigns were accepted and which were not, with the reason per campaign; accepted ones are kept.
- The restaurant can submit with nothing selected and sees the action disabled with "Selecione ao menos uma campanha".
- On a network failure mid-submit, the restaurant sees "Não conseguimos confirmar o envio." with a "Verificar status" action that reads real server state instead of resubmitting.

### F11 — Proof of minimum purchase

**Acceptance criteria**
- The approved restaurant can upload a proof (PDF, XML, JPG, PNG up to 10 MB), declare amount, purchase date and distributor, optionally the NF-e key, and sees inline validation of the 44-digit format.
- The restaurant can submit and sees "Comprovante em análise" with the file name and the deadline.
- The restaurant that sends an NF-e key already used elsewhere sees "Esta nota já foi usada em outra campanha."
- The restaurant can resend after a rejection and sees the supplier's reason above the upload field, with previous values pre-filled.
- The restaurant that declares an amount below the minimum sees a warning before submitting and can still submit — the decision belongs to the supplier.
- The restaurant that misses the deadline sees "Prazo encerrado", upload disabled, assets still locked.
- A failed upload offers a retry on that specific file and never leaves the enrollment half-submitted.

### F12 — Media Kit unlock

**Acceptance criteria**
- The restaurant below `min_order_confirmed` sees the asset grid blurred with **"Materiais liberados após a compra mínima"**, plus the exact missing step ("Falta enviar o comprovante" / "Comprovante em análise") — and no downloadable URL exists in the DOM or in any response.
- The restaurant at `min_order_confirmed` can download any asset individually or as "Baixar tudo (.zip)".
- The restaurant can request an asset from a campaign it has not confirmed and receives 403 from the edge function, seeing "Você não tem acesso a este material".
- An expired signed URL shows "Link expirado, gere novamente" with a regenerate action, never a raw storage error.
- With the enrollment confirmed but nothing uploaded yet, the restaurant sees "O fornecedor ainda não publicou os materiais."

### F13 — Offer editor

**Acceptance criteria**
- The restaurant with a confirmed enrollment can edit headline, description, terms, optional image and CTA, and sees a live preview of the public card exactly as the consumer will see it.
- The restaurant sees the supplier's suggested copy pre-filled and can always "Restaurar sugestão do fornecedor".
- The restaurant choosing WhatsApp sees phone masking; choosing iFood or site has the URL validated **server-side too**, so a bypassed client still fails.
- The restaurant can publish and sees "Publicada — vai ao ar em DD/MM"; the offer is not in the public catalog before the activation date.
- The restaurant can unpublish and sees the offer leave the public catalog on the next request.
- The restaurant cannot publish without a CTA and sees which field blocks it.
- After the campaign date, the editor is read-only with "Campanha encerrada" plus its own views and clicks.

---

### Block D — Supplier

### F14 — Brand profile

**Acceptance criteria**
- The supplier can fill brand name, legal name, CNPJ, logo, description and contacts, and sees inline validation of CNPJ check digits and of logo size/format.
- The supplier can save and sees a success toast plus the logo in the header.
- The supplier can bypass the client and submit an invalid CNPJ, and the server rejects it with a field-level error — the row is never written.
- With an incomplete profile, the supplier sees "Complete o perfil da marca para criar campanhas" linked to the missing fields, and campaign creation is blocked.

### F15 — Campaign builder

**Acceptance criteria**
- The supplier can create a campaign as draft, choose its kind (oferta or experiência), and sees it under "Rascunhos" with an "Incompleta" badge and a checklist of what is missing.
- The supplier can select target cities from the curated list as removable chips; saving with none is blocked with "Selecione ao menos uma cidade".
- The supplier sees field errors when the enrollment window closes after the activation date, when the proof deadline falls after it, or when the activation date is in the past.
- The supplier can submit for curation and sees "Em análise" with everything locked except cancel.
- The supplier sees a published campaign as "Publicada" with the enrolled count and a countdown to the activation date.
- After the first approved enrollment, the supplier sees `min_order_amount` and `activation_date` disabled with "Já existe restaurante aprovado nesta campanha".
- On publishing, the supplier can see the generated OG card preview and how the campaign will look when shared.

### F16 — Media Kit upload

**Acceptance criteria**
- The supplier can upload multiple assets (banner, post, story, print, video, PDF), sees per-file progress, and one failing file does not abort the others.
- The supplier can reorder assets by drag-and-drop, with the change reverted visibly if it fails.
- The supplier can flag exactly one asset as a public teaser and sees it on the campaign card; every other asset stays private.
- The supplier can delete an asset before publication and sees a confirmation naming the file; after publication deletion is blocked with the reason.
- With nothing uploaded, the supplier sees "Nenhum material enviado. O restaurante só recebe os materiais depois da compra mínima."

### F17 — Enrollment and proof approval

**Acceptance criteria**
- The supplier can see pending enrollments with restaurant name, city, neighbourhood, cuisine, opening date and Instagram, plus the campaign's remaining capacity.
- The supplier sees contact details only after approving an enrollment; before that the block reads "Disponível após aprovação".
- The supplier can approve or reject with a mandatory reason, and can bulk-approve with a per-row result including rows that failed because capacity ran out mid-operation.
- The supplier can open a proof, view the file inline next to the campaign minimum, and approve or reject it.
- Approving a proof below the minimum requires confirming "Valor abaixo do mínimo. Confirmar mesmo assim?" and records who overrode it.
- Approving a proof unlocks the assets for that restaurant immediately.
- An overdue enrollment is read-only for the supplier; only an admin can unblock it, and it is audited.

### F18 — Performance dashboard

**Acceptance criteria**
- The supplier can open a campaign dashboard and sees restaurants enrolled, restaurants confirmed, offer views, CTA clicks and the view→click rate.
- The supplier can filter by city and date range and sees the numbers update.
- The supplier can sort the per-restaurant table by views or clicks.
- Before the activation date, the supplier sees "Os dados aparecem após a data de ativação" alongside the enrollment counts — zeros are never presented as results.
- The supplier can never see another supplier's numbers, and a crafted request for another campaign returns no rows.

---

### Block E — Admin and cross-cutting

### F19 — Admin console

**Acceptance criteria**
- The admin can create, rename, activate and deactivate cities, and sees how many restaurants and campaigns are attached before deactivating one.
- The admin can review submitted campaigns in a preview identical to the public page, and approve or reject with a reason.
- The admin can take a published campaign down with a reason and sees its offers leave the public catalog.
- The admin can search restaurants and suppliers by name, CNPJ or city with pagination, and sees "Nenhum resultado para "{termo}"." when there is none.
- The admin can open any enrollment and see its full history — requested, decided, proof, confirmation — with timestamp and actor.
- The admin can read an audit log of every approval, rejection, suspension, takedown and manual override, and cannot edit or delete any entry.

### F20 — Design system contract

Non-negotiable: **blue is action, orange is identity.**

**Acceptance criteria**
- A reviewer can grep `src/**/*.tsx` for hex colours, `rounded-[`, and arbitrary spacing values and finds none — every colour, radius and spacing comes from a CSS variable in `index.css` mapped into Tailwind.
- A reviewer can inspect every button, link, focus ring and selected state and finds Google blue (`--blue-600` / `--blue-700` / `--blue-050`); **no orange CTA exists anywhere in the product**.
- A reviewer can find `--brand-600` used only in the logo, the **"Campanha ativa"** badge, headline underlines and highlight icons.
- A reviewer can change `--brand-600` in `index.css` and sees the whole brand shift with no other file touched.
- The user sees pill buttons (`--radius-pill`), cards at `--radius-card`, and 1px `--outline` borders instead of heavy shadows, on every screen.
- The user on mobile sees 64px between sections; on desktop 96–128px; the 8pt grid holds throughout.
- Body text never exceeds ~70 characters per line; headlines are 48–72px on desktop with tight tracking.
- No decorative gradient, no coloured shadow, no emoji used as an icon, and no row of three symmetric generic-icon cards exists in the product.

### F21 — Cross-cutting behaviour

- Every data screen implements **empty, loading, error and success** states with real Portuguese copy.
- All data access goes through `src/services/`; components never call Supabase directly.
- Every write validated by a shared zod schema reused by the corresponding edge function.
- Every destructive action confirms while naming the object affected.
- Public pages are server-rendered; private areas are client-rendered in the same build.

---

## 2. Explicitly out of v1

**From the brief:** points/gamification, full restaurant microsites, in-app chat,
payments, native mobile app, consumer event ticketing.

**From the decisions above:**
1. **Clube de Experiências** — login, saved offers, city alerts, exclusive events (A7). v1 captures emails only.
2. Multi-user teams per company (A3).
3. Chains with multiple units (A4).
4. Automated NF-e / ERP validation (A1).
5. Restaurant-chosen or recurring campaign dates (A2).
6. Vouchers, redemption codes, counter validation (A5).
7. A separate `experiences` entity with venue, capacity and RSVP (B2).

**From scoping:**
8. Map / geolocation / "perto de mim" — city filter only.
9. Consumer reviews, ratings or comments.
10. Waiting list when a campaign is full.
11. Supplier→restaurant broadcast or push notifications.
12. Asset personalisation (auto-compositing the restaurant's logo into a banner).
13. Public API, BI integrations, exports beyond a simple CSV.
14. Internationalisation — `pt-BR` only.
15. WhatsApp Business API (the CTA is a plain `wa.me` link).
16. Paid media / UTM campaign management inside the product.
17. Soft-delete/restore UI — deletions are status changes; purge is manual.
18. AMP, native app indexing, or any second rendering target.

---

## 3. Definition of done for v1

A person in São Paulo searches for something to do on a Thursday, lands on
`/ofertas/sao-paulo` from Google, sees six restaurants running the same campaign that
day with a **"Campanha ativa"** badge, taps a CTA and opens WhatsApp — while the
supplier that funded it sees those clicks in its dashboard the next morning, the
restaurant that published it got the Media Kit only after its proof was approved, and
every RLS policy has a test that attempts the forbidden read and fails.
