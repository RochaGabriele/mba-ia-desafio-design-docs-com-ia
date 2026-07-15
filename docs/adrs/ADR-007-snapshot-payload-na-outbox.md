# ADR-007 — Snapshot do payload renderizado na inserção da outbox

- **Status:** Aceito
- **Data:** 15 de julho de 2026
- **Autor:** Gabriele Rocha
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior – Plataforma), Bruno (Eng. Pleno – Pedidos) — decisão tomada no fechamento da reunião [09:52], após a saída de Marcos e Sofia [09:50].

> Escopo desta ADR: **uma única decisão** — *quando* o payload do evento é materializado (na inserção da outbox, não no envio). O contrato completo do payload (campos, headers) e o modelo de dados do `webhook_outbox` pertencem ao FDD e não são reproduzidos aqui.

---

## Contexto

Pela ADR-001, quando o status de um pedido muda, uma linha de evento é inserida na tabela `webhook_outbox` **dentro da mesma transação** de `changeStatus` (ver `src/modules/orders/order.service.ts`, `changeStatus`, L126–179 — a transação já atualiza `orders`, insere em `order_status_history` e ajusta estoque). Um worker separado (ADR-002) lê essa linha depois e dispara a chamada HTTP.

Surge, então, a questão levantada por Bruno no fechamento da reunião: o evento da outbox deve guardar **o payload já renderizado** (snapshot do estado do pedido) ou apenas o `order_id`, deixando a renderização do corpo para o instante do envio pelo worker? [09:51] Bruno.

O ponto é relevante porque existe um intervalo — potencialmente longo — entre a inserção do evento e o envio efetivo:

- O worker opera em polling e há latência mínima de processamento (ADR-002).
- Em falha, o retry com backoff pode empurrar a última tentativa para ~15h depois da primeira (ADR-003: 1m/5m/30m/2h/12h).
- Um item que cai na DLQ pode ser reprocessado **manualmente e sem prazo definido** via `POST /admin/webhooks/dead-letter/:id/replay` (ADR-003).

Durante qualquer parte dessa janela, o pedido pode sofrer novas mudanças (inclusive novas transições de status). Se o payload fosse renderizado só no envio, o evento passaria a refletir o **estado atual** do pedido — e não o estado do instante em que aquela transição ocorreu.

---

## Decisão

A linha da `webhook_outbox` guarda o **payload já renderizado (snapshot em JSON)** no momento da inserção, dentro da transação de mudança de status. O worker, ao enviar, apenas lê o snapshot persistido e o transmite — **não** reconsulta o pedido nem reconstrói o corpo no envio.

> "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito." — [09:52] Larissa. "Concordo, snapshot na inserção." — [09:52] Diego. "Beleza, snapshot. Decidido." — [09:52] Bruno.

Consequência semântica central: **o evento descreve a transição no instante em que ela aconteceu** — é imutável a partir da inserção. O snapshot é montado a partir dos dados do pedido já disponíveis na transação de `changeStatus` (campos derivados de `prisma/schema.prisma`, como `orderNumber`, `totalCents`, `status`, `customerId`). A estrutura exata do payload é definida no FDD.

---

## Alternativas Consideradas

### Renderizar na hora do envio (guardar apenas `order_id`) — **Descartada**

Nesta alternativa a outbox guardaria só a referência (`order_id` + transição), e o worker consultaria o pedido no banco no momento do dispatch para montar o corpo.

Motivos do descarte, discutidos em [09:51]–[09:52]:

- **Perda de fidelidade temporal.** O estado do pedido pode mudar entre a transição e o envio. Renderizar tarde faria o evento carregar o estado *atual*, não o do instante da mudança de status — exatamente o "caso esquisito" apontado por Larissa [09:52].
- **A janela é real e pode ser longa.** Backoff de até ~15h (ADR-003) e, sobretudo, **replay manual de DLQ sem prazo** ampliam bastante o intervalo entre inserção e entrega; reprocessar um item antigo enviaria o estado presente do pedido em vez do histórico que ele deveria representar.
- **Acoplamento indevido do worker.** Renderizar no envio obrigaria o worker a conhecer o schema de `orders` e a depender de o pedido ainda existir e ser consultável no momento do dispatch, contrariando o desacoplamento pretendido entre a mudança de status e a entrega (ADR-001/ADR-002).

Nenhuma outra alternativa foi levantada na reunião para esta decisão específica.

---

## Consequências

### Positivas

- **Evento fiel ao instante da transição.** O payload é imutável após a inserção; mudanças posteriores no pedido não contaminam eventos já emitidos.
- **Determinismo sob retry e replay.** Como a garantia é *at-least-once* (ADR-005), o mesmo evento pode ser enviado mais de uma vez (retries da ADR-003, replay de DLQ). Com snapshot, **toda reentrega carrega exatamente o mesmo corpo** — coerente com a deduplicação do cliente por `X-Event-Id` e sem risco de o payload "mudar entre as tentativas".
- **Worker mais simples e desacoplado.** O envio não faz nova query nem joins de relação no dispatch; lê a linha e posta. Reduz o custo por envio e isola o worker do schema de `orders`.
- **Sem dependência do pedido no envio.** Não há corrida leitura-após-escrita nem exigência de o pedido continuar existindo/consultável no instante do dispatch.

### Negativas

- **Redundância de armazenamento.** Cada linha da outbox carrega um JSON completo em vez de apenas um `order_id`, aumentando os bytes por registro. Mitigado por o payload ser enxuto por decisão de contrato — não inclui `items` [09:43] Diego —, pelo teto de 64KB (PRD-NFR-04) e pelo arquivamento das linhas entregues após ~30 dias (fora do escopo desta feature — [09:08] Diego).
- **Dado "envelhecido" por projeto.** Se o pedido for corrigido depois do evento (ex.: ajuste de total), o webhook entregue mostrará o valor histórico, não o atual. Isso é **intencional** — o cliente que precisa do estado corrente consulta `GET /orders/:id` depois [09:43] Diego.
- **Trabalho extra na transação de status.** Serializar o snapshot ocorre dentro da transação de `changeStatus`, que já é pesada (update em `orders`, insert em `order_status_history`, ajuste de estoque — [09:04] Bruno). Acrescenta um pequeno custo à transação.

### Trade-off explícito

Aceita-se **mais armazenamento e uma pequena sobrecarga na transação de mudança de status** em troca de **correção semântica, determinismo sob retry/replay e um worker mais simples e desacoplado**. Dado que o payload é enxuto, limitado a 64KB e as linhas entregues são arquivadas, o custo de armazenamento é considerado marginal frente ao ganho de o evento representar de forma confiável o instante da transição — inclusive quando reprocessado horas ou dias depois.

---

## Referências

- **Reunião (TRANSCRICAO.md):** [09:51] Bruno (levanta a questão); [09:52] Larissa, [09:52] Diego, [09:52] Bruno (decisão: snapshot na inserção). Contexto correlato: [09:43] Diego (payload enxuto, sem `items`; cliente consulta `GET /orders/:id` para detalhe); [09:04] Bruno (transação de status já é pesada); [09:08] Diego (arquivamento de entregues em ~30 dias, fora de escopo).
- **Código:** `src/modules/orders/order.service.ts` (`changeStatus`, L126–179 — transação onde o snapshot é inserido); `prisma/schema.prisma` (campos de origem: `orderNumber`, `totalCents`, `status`, `customerId`).
- **ADRs relacionadas:** ADR-001 (outbox no MySQL, inserção atômica na transação de status); ADR-002 (worker em processo separado, polling); ADR-003 (retry com backoff e DLQ/replay); ADR-005 (at-least-once com `X-Event-Id`). ADR-008 trata da **filtragem** de eventos na inserção (decisão distinta).
- **Fact-base:** D7 (seção 2); referências correlatas PRD-NFR-04 (limite de 64KB), PRD-NFR-05 (atomicidade/rollback), FDD-INT-01 e FDD-INT-08 (integração e modelo `webhook_outbox`). O contrato completo do payload e o modelo de dados detalhado são responsabilidade do FDD.
