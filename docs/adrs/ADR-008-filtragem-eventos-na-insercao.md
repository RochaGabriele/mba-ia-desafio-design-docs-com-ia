# ADR-008 — Filtragem de eventos por assinatura na inserção da outbox

- **Data:** 15 de julho de 2026
- **Autor:** Gabriele Rocha
- **Decisores:** Marcos (PM), Bruno (Eng. Pleno – Pedidos), Diego (Eng. Sênior – Plataforma), Larissa (Tech Lead, conduzindo)
- **Decisão coberta:** D8 (fact-base §2)

---

## Status

**Aceito** — decidido na reunião técnica de definição da feature [09:33][09:34].

---

## Contexto

Cada endpoint de webhook cadastrado por um cliente escolhe **quais status de pedido quer ouvir**, informados como uma lista (`event_types`). É o requisito **PRD-FR-05**: um cliente pode assinar, por exemplo, "só quero saber quando o pedido vira `SHIPPED` e `DELIVERED`" [09:33] Marcos. Os valores válidos dessa lista são o enum `OrderStatus` do domínio (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) — ver `prisma/schema.prisma`.

A gravação do evento acontece pelo padrão outbox (ADR-001): dentro da mesma transação de `changeStatus` que atualiza `orders` e `order_status_history`, chama-se a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que insere a linha na `webhook_outbox` usando o `tx` da transação corrente [09:41] Bruno/Diego; ver `src/modules/orders/order.service.ts` (`changeStatus`, L126–179). Cada linha inserida guarda um snapshot completo do payload (ADR-007) [09:52].

Surgiu então a pergunta de **em que momento aplicar o filtro de assinatura**: gravar todo evento de mudança de status na outbox e decidir na entrega quem recebe, ou já filtrar antes de gravar. Diego levantou explicitamente a dúvida — "filtra na inserção do outbox ou na hora de mandar?" [09:34]. Como cada evento gera uma linha (e um snapshot) na outbox, a escolha do ponto de filtragem tem impacto direto no volume da tabela — uma preocupação de performance já registrada para o worker, que lê os pendentes em batches pequenos [09:08] Diego.

---

## Decisão

**A filtragem por assinatura ocorre na inserção da outbox, não no envio.**

No momento em que `publishWebhookEvent` é chamada (dentro da transação de mudança de status), consulta-se as assinaturas dos endpoints ativos do `customer` do pedido. **Se nenhum webhook daquele customer assina o status de destino (`toStatus`), nenhuma linha é inserida na outbox.** Só há inserção quando existe ao menos um endpoint interessado naquele status [09:34] Bruno, [09:34] Diego.

Consequência semântica importante: **"nenhum assinante" é um resultado válido, não um erro.** A ausência de inserção não dispara rollback — a mudança de status do pedido segue normalmente. O rollback previsto em D-pequenas ([09:40][09:41]) aplica-se apenas quando a inserção que *deveria* acontecer falha, não quando ela simplesmente não é necessária.

O `customer_id` usado nessa consulta vem do próprio pedido (não do JWT, que é do operador) [09:32].

---

## Alternativas Consideradas

| ID | Alternativa | Por que foi descartada | Origem |
|----|-------------|------------------------|--------|
| ALT-8.1 | **Filtrar no envio** — gravar uma linha na outbox para toda mudança de status e o worker decidir, na hora de disparar, quais endpoints assinam aquele status | A outbox acumularia uma linha (com snapshot) para todo evento, mesmo quando nenhum cliente assina aquele status. Gasta linha na tabela e obriga o worker a ler e descartar eventos sem destino, contrariando a estratégia de manter a tabela enxuta e o batch pequeno | [09:34] Bruno/Diego; [09:08] Diego |

O consenso foi imediato após a proposta de Bruno ("na inserção… economiza linha na tabela"), com concordância explícita de Diego [09:34].

---

## Consequências

### Positivas

- **Economia de linhas na `webhook_outbox`:** só se materializam eventos com pelo menos um assinante. Menos crescimento da tabela e menos leitura para o worker, alinhado à preocupação de performance de [09:08] Diego.
- **Outbox mais limpa:** toda linha na outbox é, por construção, trabalho real de entrega — não há linhas "mortas" sem destino a serem varridas pelo worker.
- **Menos snapshots armazenados:** como cada linha carrega o payload renderizado (ADR-007), filtrar antes de gravar reduz também o volume de payloads persistidos.

### Negativas (trade-offs explícitos)

- **Leitura extra no caminho crítico:** a decisão de inserir passa a exigir uma consulta às assinaturas dos endpoints do customer *dentro* da transação de `changeStatus`, que já é reconhecidamente pesada (update em `orders`, insert em `order_status_history`, débito de estoque) [09:04] Bruno. **Trade-off aceito:** uma leitura adicional barata na transação em troca da economia de linhas e do trabalho do worker.
- **Filtro congelado no instante do evento:** a decisão de gravar (ou não) é tomada quando o status muda. Se um cliente cadastrar ou ampliar uma assinatura *depois* que a mudança ocorreu, o evento não gravado **não pode ser entregue retroativamente** — não existe linha na outbox para reprocessar. Na alternativa descartada (filtro no envio), uma assinatura tardia poderia alcançar um evento ainda pendente. **Trade-off aceito:** a feature entrega notificações de mudanças de status a partir do que já está assinado no instante do evento; abrir mão da entrega retroativa de eventos não gravados é uma consequência assumida do desenho (não um requisito discutido na reunião).
- **Acoplamento de `publishWebhookEvent` às assinaturas:** a função pura que recebe o `tx` precisa conhecer o modelo de endpoints (`webhook_endpoints`) e seus `event_types` para decidir a inserção, ampliando ligeiramente sua responsabilidade além de apenas "montar e gravar o evento".

---

## Referências

**Reunião (TRANSCRICAO.md):**
- [09:33] Marcos — filtro de eventos é uma lista de status que o webhook quer ouvir (PRD-FR-05).
- [09:34] Diego — levanta a dúvida: filtrar na inserção ou no envio.
- [09:34] Bruno — decide filtrar na inserção; se nenhum webhook do customer quer o status, não insere ("economiza linha na tabela").
- [09:34] Diego — concorda.
- [09:08] Diego — worker lê pendentes em batch pequeno; preocupação de performance/volume da tabela.
- [09:32] Bruno/Marcos/Larissa — `customer_id` não vem do JWT; vem do pedido/body/path.
- [09:40][09:41] Bruno/Diego — inserção da outbox dentro da transação; falha da inserção devida → rollback.
- [09:41] Bruno/Diego — assinatura `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- [09:52] Larissa/Diego/Bruno — snapshot do payload na inserção (ADR-007).

**Código:**
- `src/modules/orders/order.service.ts` (`changeStatus`, L126–179) — ponto onde `publishWebhookEvent` é chamada dentro do `$transaction`.
- `prisma/schema.prisma` — enum `OrderStatus`, domínio de valores válidos para a lista `event_types`.

**ADRs relacionados:** ADR-001 (padrão outbox no MySQL), ADR-007 (snapshot do payload na inserção). **Requisitos:** PRD-FR-05. **Integração:** FDD-INT-01.
