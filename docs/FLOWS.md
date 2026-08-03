# Campaign Hub — Fluxos por perfil

Four end-to-end journeys. Every step lists its **empty**, **loading**, **error** and
**success** states, because those are the states that get skipped when a screen is
built in a hurry. User-facing copy is Brazilian Portuguese and quoted exactly as it
must appear.

Shared conventions across all screens:

- **Loading:** skeletons matching the final layout — never a spinner in the middle of an empty page, never a layout shift when data lands.
- **Error:** a named message plus a retry action. Never an empty list implying "there is nothing here" when the request actually failed.
- **Empty:** a sentence explaining why it is empty plus the one action that fixes it.
- **Offline:** a persistent banner "Sem conexão. Suas alterações não foram salvas."
- **Session expired:** modal "Sua sessão expirou. Entre novamente." preserving the current URL for post-login redirect.

---

## 1. Consumer journey (no login)

```
Home (hero + cidade) → Catálogo da cidade → Página da campanha
   → Página/cartão do restaurante → CTA externo (WhatsApp / iFood / mapa)
                        ↓ opcional
                  Clube de Experiências
```

### 1.1 Home

Hero: **"As melhores ofertas da sua cidade, no mesmo dia."** with the city selector
immediately below. Header always shows **"Sou Restaurante"** and **"Sou Fornecedor"**.

| State | Behaviour |
|---|---|
| First visit | City selector open by default, list of active cities, "Detectar minha cidade" as a secondary option (IP-based guess, always overridable). |
| Returning visit | City read from URL `?cidade=` first, then from local storage. The URL always wins so a shared link is faithful. |
| Loading | Skeleton hero copy is static (no skeleton), offer cards are skeletons. |
| Empty (no active city in the system) | "Estamos chegando na sua região." + Clube signup. This is a launch-day state and must not look broken. |
| Error | "Não foi possível carregar as cidades." + "Tentar novamente". |

### 1.2 Catalog for a city

Cards ordered by activation date, then restaurant name. Each card: restaurant name,
neighbourhood, campaign brand, the offer headline, the date, and the CTA.

| State | Behaviour |
|---|---|
| Success | Cards render; a `offer_view` event fires once per offer per session (deduped, F13). |
| Loading | 6 skeleton cards, same grid geometry. |
| Empty — city has no live offers | "Ainda não há ofertas ativas em {cidade}." + two actions: join the Clube to be notified, and a list of the nearest cities that do have offers. |
| Empty — filter too narrow | If a campaign filter is applied: "Nenhuma oferta com esse filtro." + "Limpar filtros". |
| Error | "Não foi possível carregar as ofertas." + retry. Cards already rendered stay on screen. |
| Deactivated/invalid city in URL | Falls back to the selector with "Não encontramos essa cidade." — never a 404. |

### 1.3 Campaign page

Campaign story, activation date (with a countdown when it is in the future), and every
participating restaurant in the selected city.

| State | Behaviour |
|---|---|
| Success — upcoming | "Acontece em DD/MM" + countdown; offers visible but marked "Vai ao ar em DD/MM". |
| Success — today | "Acontecendo hoje" badge. |
| Past campaign | "Esta campanha já aconteceu." + current campaigns in the same city. Never a 404 — these URLs get shared. |
| Empty — campaign live but no restaurant in this city | "Nenhum restaurante confirmado em {cidade} ainda." + a city switcher. |
| Loading | Header skeleton + restaurant list skeleton. |
| Error / unknown slug | 404 page in Portuguese with a link to the catalog. |

### 1.4 CTA click

| State | Behaviour |
|---|---|
| Success | Opens the external destination in a new tab; `cta_click` is recorded **before** navigation using a fire-and-forget request. |
| Tracking fails | Navigation happens anyway. A dropped analytics event never blocks a conversion. |
| Malformed destination (data drift) | The CTA is hidden and the card shows "Contato indisponível no momento." — never a broken link. |
| Rate-limited / duplicate | Silently deduped server-side; the user notices nothing. |

### 1.5 Clube de Experiências (optional)

| State | Behaviour |
|---|---|
| Signup | Email + city + consent checkbox. Success: "Enviamos um link para o seu e-mail." |
| Already a member | Same confirmation message (no account enumeration). |
| Magic link expired | "Este link expirou." + resend action. |
| Save an offer while logged out | Inline signup in a sheet, the offer is saved right after confirmation — the user never loses their place. |
| Empty — "Salvos" with nothing | "Você ainda não salvou nenhuma oferta." + "Ver ofertas da minha cidade". |
| Unsubscribe | One click from the email, no login: "Você não receberá mais nossos e-mails." |
| Error | "Não foi possível concluir seu cadastro." + retry; the typed email is preserved. |

---

## 2. Restaurant journey

```
Cadastro → Análise do admin → (aprovado) → Descobrir campanhas
  → Selecionar várias → Enviar pedido único → Aguardar fornecedor
  → Comprar o mínimo → Enviar comprovante → Aguardar validação
  → min_order_confirmed → Materiais liberados + Editor da oferta
  → Publicar → Dia da ativação
```

### 2.1 Signup and approval

| State | Behaviour |
|---|---|
| Form | Name, CNPJ, city (from the curated list), address, opening date, contacts, Instagram. CNPJ check digits validated in the browser **and** on the server. |
| Duplicate CNPJ | Field error "Este CNPJ já está cadastrado." + "Entrar" link. |
| Submitted | Full-screen state "Cadastro em análise" explaining the expected time and what happens next. No dashboard is reachable. |
| Rejected | The admin's reason is shown verbatim + "Editar cadastro"; editing returns the account to `pending`. |
| Suspended | "Sua conta está suspensa." + support contact. Offers already published disappear from the public catalog. |
| Error on submit | Inline errors per field; nothing is lost, the form keeps every value. |

### 2.2 Discover campaigns

| State | Behaviour |
|---|---|
| Success | Cards for campaigns targeting the restaurant's city, with brand, activation date, minimum purchase and enrolled count. |
| Empty — no campaigns in the city | "Ainda não há campanhas para a sua cidade." + "Avise-me quando houver". |
| Empty — already joined everything | "Você já participa de todas as campanhas disponíveis." + link to "Minhas campanhas". |
| Full campaign | Badge "Vagas esgotadas", CTA disabled, card kept visible (it is still social proof). |
| Enrollment window closed | Badge "Inscrições encerradas". |
| Already enrolled | **"Participar da campanha"** is replaced by the current status ("Aguardando aprovação", "Enviar comprovante", "Confirmado"). |
| Loading | Skeleton cards. |
| Error | "Não foi possível carregar as campanhas." + retry. |

### 2.3 Multi-select and single order

| State | Behaviour |
|---|---|
| Selecting | A persistent "Sua seleção" summary shows each campaign, its minimum purchase and the total. Removable per item. |
| Empty selection | The submit button is disabled with "Selecione ao menos uma campanha". |
| Submitting | Button in loading state, the whole selection locked to prevent double submission. |
| Full success | "Pedido enviado" listing every campaign as "Aguardando aprovação do fornecedor". |
| **Partial success** | Result screen splits accepted vs not accepted, naming the reason per campaign ("Vagas esgotadas", "Inscrições encerradas", "Você já participa"). The accepted ones are kept — the order is never rolled back wholesale. |
| Total failure | "Nenhuma campanha pôde ser enviada." with per-campaign reasons and the selection preserved. |
| Network error mid-submit | "Não conseguimos confirmar o envio." + "Verificar status", which reloads the real server state instead of resubmitting. |

### 2.4 Waiting for the supplier

| State | Behaviour |
|---|---|
| Pending | "Aguardando aprovação do fornecedor" + when it was requested. |
| Approved | Prominent next step: "Faça a compra mínima e envie o comprovante" with the amount, the description ("10 barris de 30L") and the deadline. |
| Rejected | The supplier's reason verbatim + other campaigns in the same city. |
| Cancelled by the restaurant | Confirmation modal naming the campaign; afterwards the campaign returns to the discovery list. |

### 2.5 Proof of minimum purchase

| State | Behaviour |
|---|---|
| Form | File (PDF/XML/JPG/PNG, up to 10 MB), declared amount, purchase date, distributor, optional 44-digit NF-e key. |
| File too large / wrong type | Inline error on the file row: "Arquivo acima de 10 MB." / "Formato não aceito. Envie PDF, XML, JPG ou PNG." |
| Invalid NF-e key | "A chave da NF-e deve ter 44 dígitos." |
| Duplicate NF-e key | Server error surfaced as "Esta nota já foi usada em outra campanha." |
| Upload in progress | Per-file progress bar; navigation away asks for confirmation. |
| Upload failed | Retry on that specific file. The enrollment is never left half-submitted. |
| Submitted | "Comprovante em análise" + file name + deadline. |
| Rejected | The supplier's reason above the upload field, previous values pre-filled, a new file may be sent while the deadline holds. |
| Deadline passed | "Prazo encerrado" — upload disabled, assets stay locked, campaign moves to history. |
| Amount below the minimum | Warning before submitting: "O valor declarado está abaixo da compra mínima. O fornecedor pode recusar." Submission is still allowed — the decision is the supplier's. |

### 2.6 Media Kit unlock

| State | Behaviour |
|---|---|
| Locked | Blurred grid with **"Materiais liberados após a compra mínima"** and the exact missing step underneath ("Falta enviar o comprovante" / "Comprovante em análise"). No downloadable URL exists in the DOM or in any response. |
| Unlocked | Full grid by kind, individual download and "Baixar tudo (.zip)". |
| Expired signed link | "Link expirado, gere novamente" + a regenerate action. |
| Download error | Per-file error row; the other files remain downloadable. |
| Empty — confirmed but the supplier uploaded nothing | "O fornecedor ainda não publicou os materiais." + "Avisar o fornecedor". |

### 2.7 Offer editor

| State | Behaviour |
|---|---|
| Initial | Pre-filled with the supplier's suggested copy; "Restaurar sugestão do fornecedor" always available. |
| Live preview | The public card rendered exactly as the consumer will see it, updating as you type. |
| Validation | Headline ≤ 80 chars, description ≤ 400, both with visible counters. WhatsApp masked and validated; URLs validated server-side too. |
| Missing CTA | Publishing blocked with "Escolha como o cliente vai pedir ou reservar." |
| Published | "Publicada — vai ao ar em DD/MM" (or "no ar agora" on the day). |
| Unpublished | "Oferta fora do ar" + republish action; disappears from the public catalog on the next request. |
| Save error | Toast "Não foi possível salvar." — nothing typed is lost. |
| Campaign day passed | Editor becomes read-only with "Campanha encerrada" + the restaurant's own numbers (views, clicks). |

---

## 3. Supplier journey

```
Cadastro → Análise do admin → Perfil da marca → Criar campanha
  → Enviar para curadoria → Publicada → Aprovar restaurantes
  → Validar comprovantes → Acompanhar desempenho
```

### 3.1 Signup, approval, brand profile

| State | Behaviour |
|---|---|
| Pending | "Cadastro em análise" — the campaign builder is unreachable. |
| Rejected / suspended | Reason verbatim + edit action; suspension pulls its published campaigns from the public catalog. |
| Incomplete profile | "Complete o perfil da marca para criar campanhas" linking to the missing fields. |
| Logo upload error | "Não foi possível enviar a imagem." + retry; the rest of the form is untouched. |
| Success | Toast + logo visible in the header. |

### 3.2 Campaign builder

| State | Behaviour |
|---|---|
| Empty list | "Você ainda não criou nenhuma campanha." + "Criar campanha". |
| Draft | Badge "Incompleta" until required fields are filled; a checklist shows exactly what is missing. |
| No cities selected | "Selecione ao menos uma cidade". |
| Invalid dates | Field errors: enrollment must close on or before the activation date; the proof deadline cannot fall after the activation date; the activation date must be in the future. |
| Submitted for curation | "Em análise" — everything locked except cancel. |
| Rejected by admin | Reason verbatim, campaign returns to `draft`, editing reopens. |
| Published | "Publicada" + enrolled count + countdown. |
| Frozen fields | After the first approved enrollment, `min_order_amount` and `activation_date` are disabled with "Já existe restaurante aprovado nesta campanha". |
| Error saving | Toast + retry; nothing is lost. |

### 3.3 Media Kit upload

| State | Behaviour |
|---|---|
| Empty | "Nenhum material enviado. O restaurante só recebe os materiais depois da compra mínima." |
| Uploading | Per-file progress; one failing file does not abort the others. |
| Per-file error | "Falha ao enviar {arquivo}." + retry on that row. |
| Reorder | Drag-and-drop with an optimistic update, reverted on failure. |
| Delete | Confirmation naming the file; blocked once the campaign is published, with the reason shown. |
| Public preview | One asset may be flagged as a teaser; the flag is exclusive and the UI says so. |

### 3.4 Enrollment approval

| State | Behaviour |
|---|---|
| Empty | "Nenhum restaurante aguardando aprovação." |
| List | Restaurant name, city, neighbourhood, cuisine, opening date, Instagram, and the campaign's remaining capacity. Contacts show "Disponível após aprovação". |
| Approve | Row moves to "Aprovadas"; the restaurant is notified and now owes a proof. |
| Reject | Mandatory reason; without it the action is blocked. |
| Bulk approve | Per-row result, including "Vagas esgotadas" for rows that failed mid-operation. |
| Capacity exhausted | Banner "Todas as vagas foram preenchidas." and remaining pending rows can only be rejected. |
| Error | Per-row error, the list is not reloaded from scratch, decisions already made are preserved. |

### 3.5 Proof validation

| State | Behaviour |
|---|---|
| Empty | "Nenhum comprovante aguardando validação." |
| Review | File rendered inline (PDF/image), declared amount, purchase date, distributor, NF-e key, and the campaign minimum side by side. |
| Approve | The enrollment becomes `min_order_confirmed`, the assets unlock immediately, and the restaurant is notified. |
| Approve below the minimum | Confirmation "Valor abaixo do mínimo. Confirmar mesmo assim?" — the override and its author are recorded. |
| Reject | Mandatory reason, shown to that restaurant only; it may resubmit while the deadline holds. |
| Unreadable file | Reject with the reason; there is no "pending forever" state. |
| Overdue | Read-only "Prazo encerrado" — the supplier can no longer confirm it (only an admin can, and it is audited). |

### 3.6 Performance dashboard

| State | Behaviour |
|---|---|
| Before the activation date | "Os dados aparecem após a data de ativação" + enrolled/confirmed counts, which are meaningful already. Zeros are never presented as results. |
| Live | Restaurants enrolled, restaurants confirmed, offer views, CTA clicks, view→click rate. Filters by city and date range. |
| Per-restaurant table | Sortable by views or clicks; empty rows read "Sem dados ainda". |
| Empty — no campaign yet | "Crie sua primeira campanha para ver resultados aqui." |
| Error | "Não foi possível carregar os dados." + retry; already-loaded cards stay. |
| Isolation | A crafted request for another supplier's campaign returns no rows (verification matrix #7, #8). |

---

## 4. Admin journey

```
Login → Fila de cadastros → Curadoria de campanhas → Cidades → Busca/suporte → Auditoria
```

### 4.1 Account queue

| State | Behaviour |
|---|---|
| Empty | "Nenhum cadastro aguardando análise." |
| List | Suppliers and restaurants with submission date, city and CNPJ; separate tabs, single counter in the sidebar. |
| Approve | Row moves to "Aprovados"; the owner's next login lands on their working dashboard. |
| Reject | Mandatory reason, shown verbatim to the owner. |
| Suspend | Confirmation naming the account and its consequence ("As ofertas deste restaurante sairão do ar."). |
| Error | Per-row error; the queue is not lost. |

### 4.2 Campaign curation

| State | Behaviour |
|---|---|
| Empty | "Nenhuma campanha aguardando curadoria." |
| Review | Full campaign preview exactly as the restaurant and the consumer will see it, plus the asset list. |
| Approve | Status becomes `published`, visible to restaurants in the selected cities. |
| Reject | Mandatory reason; the campaign returns to `draft` on the supplier's side. |
| Takedown | An already-published campaign can be moved to `closed` with a reason; its offers leave the public catalog. Always audited. |

### 4.3 Cities

| State | Behaviour |
|---|---|
| Empty | "Nenhuma cidade cadastrada." + "Adicionar cidade" — this is the day-one state and must be inviting, not broken. |
| Create | Name + UF, slug generated and editable; duplicates blocked with "Esta cidade já existe." |
| Deactivate | Shows how many restaurants and campaigns are attached before confirming; deactivating hides the city from the public selector but never deletes data. |
| Delete | Not offered. Deactivation only. |

### 4.4 Search and support

| State | Behaviour |
|---|---|
| Search | By name, CNPJ or city, paginated. |
| No results | "Nenhum resultado para "{termo}"." + "Limpar busca". |
| Enrollment detail | Full history — requested, decided, proof, confirmation — with timestamp and actor for each step. |
| Manual unblock | An admin may confirm an expired enrollment; a reason is mandatory and the action is written to the audit log. |
| Error | "Não foi possível carregar." + retry. |

### 4.5 Audit

| State | Behaviour |
|---|---|
| List | Every approval, rejection, suspension, takedown and manual override, with actor, action, entity and reason. |
| Empty | "Nenhuma ação registrada ainda." |
| Filter | By actor, entity and date range. |
| Immutability | Read-only in the UI and in the policy — there is no edit or delete path for anyone, including admins. |

---

## 5. Cross-journey error states

| Situation | What the user sees |
|---|---|
| Role mismatch on a URL | 403 page in Portuguese with a link back to the user's own area. Never a blank screen, never the protected data. |
| Account suspended mid-session | The next request returns 403 and the session drops to the "conta suspensa" screen. |
| Supabase unreachable | Global banner "Estamos com instabilidade. Tente novamente em instantes." Cached screens remain readable. |
| Signed URL expired | "Link expirado, gere novamente" + regenerate. |
| Concurrent edit (two tabs) | "Esta campanha foi alterada em outra aba." + reload action; no silent overwrite. |
| Server validation rejects what the client accepted | Field-level error mapped from the server response — the client never claims success on a rejected write. |
