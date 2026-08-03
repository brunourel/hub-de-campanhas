# BORA — Fluxos por perfil

Four end-to-end journeys. Every step lists its **empty**, **loading**, **error** and
**success** states, because those are the states that get skipped when a screen is
built in a hurry. User-facing copy is Brazilian Portuguese, quoted exactly as it must
appear.

Shared conventions:

- **Loading:** skeletons matching the final layout. Public pages are server-rendered
  (B1), so their first paint has content — skeletons exist only for client-side
  transitions and for private areas.
- **Error:** a named message plus a retry. Never an empty list implying "there is
  nothing here" when the request actually failed.
- **Empty:** a sentence explaining why it is empty plus the one action that fixes it.
- **Offline:** persistent banner "Sem conexão. Suas alterações não foram salvas."
- **Session expired:** modal "Sua sessão expirou. Entre novamente.", preserving the
  current URL for the post-login redirect.
- **HTTP status is part of the state.** An unknown slug is a real 404, a renamed slug
  is a real 301, a finished campaign is a 200 — never a soft 404 rendered as content.

---

## 1. Consumer journey (no login — the front door)

```
Busca no Google / link no WhatsApp
        ↓
   /ofertas/:cidade   ←→   /experiencias/:cidade
        ↓
   /campanha/:slug  →  /restaurante/:slug
        ↓
   CTA externo (WhatsApp / iFood / mapa)
        ↓ opcional
   "Avise-me" (captura de e-mail — o Clube é v2)
```

### 1.1 Home `/`

Hero: **"As melhores ofertas da sua cidade, no mesmo dia."** with the city selector
right below. Header always shows **"Sou Restaurante"** and **"Sou Fornecedor"** — blue,
never orange.

| State | Behaviour |
|---|---|
| First visit | City selector open, list of active cities, "Detectar minha cidade" as a secondary option (IP guess, always overridable). |
| Returning visit | City read from the URL first, then from local storage. The URL always wins so a shared link is faithful. |
| Loading | Server-rendered: hero and city list arrive in the HTML. No skeleton. |
| Empty (no active city) | "Estamos chegando na sua região." + email capture. This is the launch-day state and must not look broken. |
| Error | Static shell renders with "Não foi possível carregar as cidades." + retry. |

### 1.2 City catalog `/ofertas/:cidade`

| State | Behaviour |
|---|---|
| Success | Offer cards ordered by activation date, then restaurant name. Today's offers carry the **"Campanha ativa"** badge in orange — the only orange on the card. Upcoming ones read "Acontece em DD/MM". |
| No JavaScript | The full list is present in the HTML source (F1). |
| Loading (client transition) | Six skeleton cards on the same grid geometry, no layout shift. |
| Empty — no live offers | "Ainda não há ofertas ativas em {cidade}." + "Avise-me quando houver ofertas em {cidade}" + the nearest cities that do have offers. |
| Empty — filter too narrow | "Nenhuma oferta com esse filtro." + "Limpar filtros". |
| Unknown or deactivated city | **404 status** with the city selector and "Não encontramos essa cidade." |
| Renamed city slug | **301** to the current URL, transparent to the visitor. |
| Error | "Não foi possível carregar as ofertas." + retry; anything already rendered stays. |
| Tracking | One `city_view`, and one `offer_view` per offer per session per day. |

### 1.3 Campaign page `/campanha/:slug`

| State | Behaviour |
|---|---|
| Success — upcoming | "Acontece em DD/MM" + countdown; offers listed as "Vai ao ar em DD/MM". |
| Success — today | **"Campanha ativa"** badge and the participating restaurants in the selected city. |
| Past campaign | "Esta campanha já aconteceu." + what is live now. **Status 200 and `noindex`** — the link is already shared, it must not 404 (A16). |
| Empty — no restaurant in this city | "Nenhum restaurante confirmado em {cidade} ainda." + a city switcher. |
| City switch | Participating restaurants change; the campaign context and the URL's canonical stay intact. |
| Renamed slug | **301** to the current URL. |
| Unknown slug | **404** page in Portuguese with links to the live catalog. |
| Share | The OG card composed at publish time (B3): campaign cover, title, city, brand. |

### 1.4 Restaurant page `/restaurante/:slug`

| State | Behaviour |
|---|---|
| Success | Name, neighbourhood, city, cuisine, cover, Instagram, live offers, past campaigns. `Restaurant` JSON-LD. |
| No live offer | "Sem ofertas ativas no momento." + history + a link to the city catalog. Status 200. |
| Never published an offer | **404** — the page does not exist yet and the slug is not in the sitemap (B4). |
| Suspended restaurant | 404 and removal from the sitemap on the next regeneration. |
| Error | "Não foi possível carregar este restaurante." + retry. |

### 1.5 Experiences `/experiencias/:cidade`

| State | Behaviour |
|---|---|
| Success | Only `kind = 'experiencia'` campaigns in that city, with date and participating venues. `Event` JSON-LD. |
| Empty | "Ainda não há experiências em {cidade}." + the offers catalog for the same city. |
| Canonical | Self-referencing; it never competes with `/ofertas/:cidade` in the index. |

### 1.6 CTA click

| State | Behaviour |
|---|---|
| Success | Opens the external destination in a new tab; `cta_click` is sent fire-and-forget **before** navigating. |
| Tracking fails | Navigation happens anyway. A dropped analytics event never blocks a conversion. |
| Malformed destination | The CTA is hidden and the card reads "Contato indisponível no momento." — never a broken link. |
| Rate-limited / duplicate | Silently deduped server-side; the visitor notices nothing. |

### 1.7 "Avise-me" email capture (the Clube is v2 — A7)

| State | Behaviour |
|---|---|
| Form | Email + the city already in context + consent checkbox. |
| Success | "Pronto! Avisamos você quando houver ofertas em {cidade}." |
| Already registered | The same message — no account enumeration, no duplicate row (`unique (lower(email), city_id)`). |
| Invalid email | Inline "Digite um e-mail válido." |
| Error | "Não foi possível concluir seu cadastro." + retry, with the typed email preserved. |
| Explicitly not built in v1 | Login, saved offers, alert emails, exclusive events. The screen never promises them. |

---

## 2. Restaurant journey

```
Cadastro → Análise do admin → (aprovado) → Descobrir campanhas
  → Selecionar várias → Enviar pedido único → Aguardar fornecedor
  → Comprar o mínimo → Enviar comprovante → Aguardar validação
  → min_order_confirmed → Materiais liberados + Editor da oferta
  → Publicar → Dia da ativação → Página pública nasce
```

### 2.1 Signup and approval

| State | Behaviour |
|---|---|
| Form | Name, CNPJ, city from the curated list, address, opening date, contacts, Instagram. CNPJ check digits validated in the browser **and** on the server. |
| Duplicate CNPJ | "Este CNPJ já está cadastrado." + an "Entrar" link. |
| Submitted | Full-screen "Cadastro em análise" explaining the expected time and what comes next. No dashboard is reachable. |
| Rejected | The admin's reason verbatim + "Editar cadastro"; editing returns the account to `pending`. |
| Suspended | "Sua conta está suspensa." + support contact. Published offers leave the public catalog and the restaurant page 404s. |
| Error on submit | Field-level errors; the form keeps every value. |

### 2.2 Discover campaigns

| State | Behaviour |
|---|---|
| Success | Cards for campaigns targeting the restaurant's city: brand, activation date, minimum purchase, enrolled count. |
| Empty — none in the city | "Ainda não há campanhas para a sua cidade." + "Avise-me quando houver". |
| Empty — all joined | "Você já participa de todas as campanhas disponíveis." + link to "Minhas campanhas". |
| Full campaign | "Vagas esgotadas", action disabled, card kept visible (it is still social proof). |
| Window closed | "Inscrições encerradas". |
| Already enrolled | **"Participar da campanha"** replaced by the current status. |
| Error | "Não foi possível carregar as campanhas." + retry. |

### 2.3 Multi-select and single order

| State | Behaviour |
|---|---|
| Selecting | A persistent "Sua seleção" panel: each campaign, its minimum purchase, the total. Removable per item. |
| Nothing selected | Submit disabled with "Selecione ao menos uma campanha". |
| Submitting | Button in loading state, the whole selection locked against double submission. |
| Full success | "Pedido enviado", every campaign listed as "Aguardando aprovação do fornecedor". |
| **Partial success** | Accepted and refused split apart, with the reason per campaign ("Vagas esgotadas", "Inscrições encerradas", "Você já participa"). Accepted ones are kept — the order is never rolled back wholesale. |
| Total failure | "Nenhuma campanha pôde ser enviada." with per-campaign reasons and the selection preserved. |
| Network error mid-submit | "Não conseguimos confirmar o envio." + "Verificar status", which reads real server state instead of resubmitting. |

### 2.4 Waiting for the supplier

| State | Behaviour |
|---|---|
| Pending | "Aguardando aprovação do fornecedor" + when it was requested. |
| Approved | Prominent next step: "Faça a compra mínima e envie o comprovante", with the amount, the description ("10 barris de 30L") and the deadline. |
| Rejected | The supplier's reason verbatim + other campaigns in the same city. |
| Cancelling | Confirmation modal naming the campaign; afterwards it returns to the discovery list. |

### 2.5 Proof of minimum purchase

| State | Behaviour |
|---|---|
| Form | File (PDF/XML/JPG/PNG ≤ 10 MB), declared amount, purchase date, distributor, optional 44-digit NF-e key. |
| File too large / wrong type | "Arquivo acima de 10 MB." / "Formato não aceito. Envie PDF, XML, JPG ou PNG." |
| Invalid NF-e key | "A chave da NF-e deve ter 44 dígitos." |
| Duplicate NF-e key | "Esta nota já foi usada em outra campanha." |
| Uploading | Per-file progress; leaving the page asks for confirmation. |
| Upload failed | Retry on that specific file. The enrollment is never left half-submitted. |
| Submitted | "Comprovante em análise" + file name + deadline. |
| Rejected | The supplier's reason above the upload field, previous values pre-filled, resend allowed while the deadline holds. |
| Deadline passed | "Prazo encerrado" — upload disabled, assets stay locked, the campaign moves to history. |
| Amount below the minimum | Warning before submitting: "O valor declarado está abaixo da compra mínima. O fornecedor pode recusar." Submission still allowed — the decision is the supplier's. |

### 2.6 Media Kit unlock

| State | Behaviour |
|---|---|
| Locked | Blurred grid with **"Materiais liberados após a compra mínima"** and the exact missing step underneath ("Falta enviar o comprovante" / "Comprovante em análise"). No downloadable URL exists in the DOM or in any response. |
| Unlocked | Full grid by kind, individual download and "Baixar tudo (.zip)". |
| Expired signed link | "Link expirado, gere novamente" + regenerate. |
| Download error | Per-file error row; the other files stay downloadable. |
| Empty — confirmed, nothing uploaded | "O fornecedor ainda não publicou os materiais." + "Avisar o fornecedor". |

### 2.7 Offer editor

| State | Behaviour |
|---|---|
| Initial | Pre-filled with the supplier's suggested copy; "Restaurar sugestão do fornecedor" always available. |
| Live preview | The public card rendered exactly as the consumer will see it, updating as you type. |
| Validation | Headline ≤ 80 chars, description ≤ 400, visible counters. WhatsApp masked; URLs validated server-side too. |
| Missing CTA | Publishing blocked with "Escolha como o cliente vai pedir ou reservar." |
| Published — first time | "Publicada — vai ao ar em DD/MM" **plus** "Sua página em BORA já está no ar: bora.app/restaurante/{slug}" — the moment B4 fires and the public page is born. |
| Published — subsequent | "Publicada — vai ao ar em DD/MM". |
| Unpublished | "Oferta fora do ar" + republish; it leaves the public catalog on the next request. |
| Save error | "Não foi possível salvar." — nothing typed is lost. |
| After the campaign date | Read-only "Campanha encerrada" + its own views and clicks. |

---

## 3. Supplier journey

```
Cadastro → Análise do admin → Perfil da marca → Criar campanha
  → Enviar para curadoria → Publicada (+ card OG gerado) → Aprovar restaurantes
  → Validar comprovantes → Acompanhar desempenho
```

### 3.1 Signup, approval, brand profile

| State | Behaviour |
|---|---|
| Pending | "Cadastro em análise" — the campaign builder is unreachable. |
| Rejected / suspended | Reason verbatim + edit action; suspension pulls published campaigns from the public catalog. |
| Incomplete profile | "Complete o perfil da marca para criar campanhas", linked to the missing fields. |
| Logo upload error | "Não foi possível enviar a imagem." + retry; the rest of the form is untouched. |
| Success | Toast + logo in the header. |

### 3.2 Campaign builder

| State | Behaviour |
|---|---|
| Empty list | "Você ainda não criou nenhuma campanha." + "Criar campanha". |
| Kind | "Oferta" or "Experiência" chosen at creation; experiences render `Event` data publicly and the builder says so. |
| Draft | "Incompleta" badge with a checklist of exactly what is missing. |
| No city selected | "Selecione ao menos uma cidade". |
| Invalid dates | Enrollment must close on or before the activation date; the proof deadline cannot fall after it; the activation date must be in the future. |
| Submitted for curation | "Em análise", everything locked except cancel. |
| Rejected by admin | Reason verbatim, campaign back to `draft`, editing reopens. |
| Published | "Publicada" + enrolled count + countdown + a preview of the generated OG card as it will look when shared. |
| Frozen fields | After the first approved enrollment, `min_order_amount` and `activation_date` are disabled with "Já existe restaurante aprovado nesta campanha". |
| OG generation failed | Non-blocking warning "Não foi possível gerar o card de compartilhamento." + regenerate; the campaign stays published and falls back to the cover image. |
| Error saving | Toast + retry; nothing is lost. |

### 3.3 Media Kit upload

| State | Behaviour |
|---|---|
| Empty | "Nenhum material enviado. O restaurante só recebe os materiais depois da compra mínima." |
| Uploading | Per-file progress; one failing file does not abort the others. |
| Per-file error | "Falha ao enviar {arquivo}." + retry on that row. |
| Reorder | Drag-and-drop, optimistic, visibly reverted on failure. |
| Delete | Confirmation naming the file; blocked after publication, with the reason shown. |
| Public teaser | Exactly one asset may be flagged; the UI states the flag is exclusive. |

### 3.4 Enrollment approval

| State | Behaviour |
|---|---|
| Empty | "Nenhum restaurante aguardando aprovação." |
| List | Name, city, neighbourhood, cuisine, opening date, Instagram, remaining capacity. Contacts read "Disponível após aprovação". |
| Approve | Row moves to "Aprovadas"; the restaurant is notified and now owes a proof. |
| Reject | Mandatory reason; without it the action is blocked. |
| Bulk approve | Per-row result, including "Vagas esgotadas" for rows that failed mid-operation. |
| Capacity exhausted | Banner "Todas as vagas foram preenchidas."; remaining pending rows can only be rejected. |
| Error | Per-row error; the list is not reloaded from scratch and decisions already made are preserved. |

### 3.5 Proof validation

| State | Behaviour |
|---|---|
| Empty | "Nenhum comprovante aguardando validação." |
| Review | File rendered inline next to declared amount, purchase date, distributor, NF-e key and the campaign minimum. |
| Approve | The enrollment becomes `min_order_confirmed`, assets unlock immediately, the restaurant is notified. |
| Approve below the minimum | "Valor abaixo do mínimo. Confirmar mesmo assim?" — the override and its author are recorded. |
| Reject | Mandatory reason, visible to that restaurant only; it may resubmit while the deadline holds. |
| Unreadable file | Reject with the reason. There is no "pending forever" state. |
| Overdue | Read-only "Prazo encerrado" — only an admin can unblock it, and it is audited. |

### 3.6 Performance dashboard

| State | Behaviour |
|---|---|
| Before the activation date | "Os dados aparecem após a data de ativação" + enrolled/confirmed counts, which already mean something. Zeros are never presented as results. |
| Live | Restaurants enrolled, restaurants confirmed, offer views, CTA clicks, view→click rate. Filters by city and date range. |
| Per-restaurant table | Sortable by views or clicks; rows with nothing read "Sem dados ainda". |
| Empty — no campaign | "Crie sua primeira campanha para ver resultados aqui." |
| Error | "Não foi possível carregar os dados." + retry; loaded cards stay. |
| Isolation | A crafted request for another supplier's campaign returns no rows (matrix #7, #8). |

---

## 4. Admin journey

```
Login → Fila de cadastros → Curadoria de campanhas → Cidades → Busca/suporte → Auditoria
```

### 4.1 Account queue

| State | Behaviour |
|---|---|
| Empty | "Nenhum cadastro aguardando análise." |
| List | Suppliers and restaurants with submission date, city and CNPJ; separate tabs, one counter in the sidebar. |
| Approve | Moves to "Aprovados"; the owner's next login lands on their dashboard. |
| Reject | Mandatory reason, shown verbatim to the owner. |
| Suspend | Confirmation naming the account and the consequence: "As ofertas deste restaurante sairão do ar e a página pública deixará de existir." |
| Error | Per-row error; the queue is not lost. |

### 4.2 Campaign curation

| State | Behaviour |
|---|---|
| Empty | "Nenhuma campanha aguardando curadoria." |
| Review | A preview identical to the public page, plus the asset list and the OG card. |
| Approve | Status becomes `published`, the OG image is generated, the sitemap is refreshed, restaurants in the target cities see it. |
| Reject | Mandatory reason; the campaign returns to `draft` on the supplier's side. |
| Takedown | A published campaign moves to `closed` with a reason; its offers leave the catalog, the page stays online as history. Always audited. |

### 4.3 Cities

| State | Behaviour |
|---|---|
| Empty | "Nenhuma cidade cadastrada." + "Adicionar cidade" — the day-one state, and it must be inviting rather than broken. |
| Create | Name + UF; slug generated and editable; duplicates blocked with "Esta cidade já existe." |
| Rename slug | The old slug is kept as a 301 automatically; the admin is told so. |
| Deactivate | Shows how many restaurants and campaigns are attached before confirming; the city leaves the public selector and the sitemap but no data is deleted. |
| Delete | Not offered. Deactivation only. |

### 4.4 Search and support

| State | Behaviour |
|---|---|
| Search | By name, CNPJ or city, paginated. |
| No results | "Nenhum resultado para "{termo}"." + "Limpar busca". |
| Enrollment detail | Full history — requested, decided, proof, confirmation — with timestamp and actor. |
| Manual unblock | An admin may confirm an expired enrollment; the reason is mandatory and the action is audited. |
| Error | "Não foi possível carregar." + retry. |

### 4.5 Audit

| State | Behaviour |
|---|---|
| List | Every approval, rejection, suspension, takedown and override, with actor, action, entity and reason. |
| Empty | "Nenhuma ação registrada ainda." |
| Filter | By actor, entity and date range. |
| Immutability | Read-only in the UI **and** in the policy — no edit or delete path exists for anyone, including admins. |

---

## 5. Cross-journey error states

| Situation | What the user sees |
|---|---|
| Role mismatch on a URL | 403 page in Portuguese with a link back to the user's own area. Never a blank screen, never the protected data. |
| Account suspended mid-session | The next request returns 403 and the session drops to the "conta suspensa" screen. |
| Supabase unreachable | Public pages serve the last successful render where possible; otherwise a static shell with "Estamos com instabilidade. Tente novamente em instantes." |
| SSR render fails | A real 500 page in Portuguese with the header and a link to the home page — never a stack trace, never a white screen. |
| Signed URL expired | "Link expirado, gere novamente" + regenerate. |
| Concurrent edit in two tabs | "Esta campanha foi alterada em outra aba." + reload. No silent overwrite. |
| Server rejects what the client accepted | Field-level errors mapped from the server response. The client never claims success on a rejected write. |
