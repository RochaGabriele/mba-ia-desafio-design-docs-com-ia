# ADR-001 — Padrão Outbox no MySQL com transação atômica

| Campo | Valor |
|-------|-------|
| **Status** | Aceito |
| **Data** | 15 de julho de 2026 |
| **Autor** | Gabriele Rocha |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior – Plataforma), Bruno (Eng. Pleno – Pedidos) |
| **Contexto de negócio** | Marcos (PM) |
| **Decisão** | D1 |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |

---

## Status

**Aceito.** Decisão fechada na reunião de arquitetura da feature — *"Tá decidido então: outbox em MySQL."* [09:08] Larissa.

---

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados quando o status dos pedidos deles muda na plataforma; hoje fazem *polling* caro no `GET /orders` [09:00] Marcos. O núcleo funcional da feature é emitir um evento *outbound* toda vez que o status de um pedido muda ([09:06] Diego; PRD-FR-07).

A primeira pergunta arquitetural levantada foi: disparar a notificação **sincronamente** dentro do serviço de pedidos, ou registrar o evento em algum tipo de fila/outbox? [09:03] Larissa.

Dois problemas tornam o disparo síncrono inviável:

1. **A transação de mudança de status já é pesada.** Ela atualiza `orders`, insere em `order_status_history` e movimenta o estoque dos itens do pedido — [09:04] Bruno. Isso é confirmado no código: `changeStatus` executa tudo dentro de `this.prisma.$transaction`, chamando `tx.order.update`, `tx.orderStatusHistory.create` e o débito/reposição de estoque (`src/modules/orders/order.service.ts`, L126-179). Um HTTP call no meio dessa transação faria um cliente lento travar a mudança de status de **outros** pedidos.
2. **Não há como reconciliar falha do cliente com a transação de negócio.** Se o cliente estiver fora do ar, não é aceitável dar *rollback* na mudança de status por causa disso — [09:04] Bruno.

Ao mesmo tempo, existe um requisito de consistência forte: **não pode haver caso de o status mudar e o evento não sair** ([09:40] Bruno, [09:41] Diego; PRD-NFR-05). Ou seja, o registro do evento precisa ser atômico em relação à mudança de status.

O time é pequeno e a stack de produção já é MySQL acessado via Prisma, com ids `uuid @db.Char(36)` (`prisma/schema.prisma`). Qualquer solução que exija subir nova infraestrutura pesa desproporcionalmente para o tamanho do time [09:07] Diego.

---

## Decisão

Adotar o **padrão Outbox sobre o MySQL já existente**. Dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`, inserir uma linha numa tabela `webhook_outbox` com o evento — [09:06] Diego.

Consequência direta da atomicidade:

- Se a transação principal **commitou**, o evento foi registrado.
- Se a transação deu **rollback**, o evento some junto.
- Se a inserção na outbox **falhar** dentro da transação, a transação inteira sofre *rollback* — o status não muda sem o evento sair ([09:40] Bruno, [09:41] Diego).

Não há inconsistência possível entre "status mudou" e "evento registrado" — [09:06] Diego.

**Integração no código.** A inserção entra na transação já existente de `changeStatus` (`src/modules/orders/order.service.ts`, L126-179), após `tx.orderStatusHistory.create`, por meio de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação atual — sem injetar um repository inteiro no `OrderService` ([09:41] Bruno/Diego). O contrato detalhado dessa integração é responsabilidade do FDD (FDD-INT-01).

**Modelo de dados (alto nível).** `webhook_outbox` é um novo model no `prisma/schema.prisma`, seguindo as convenções do projeto — id `uuid @db.Char(36)` ([09:51] Larissa) e `@@map` — e modelado para leitura eficiente pelo worker (índices em status e `created_at`, [09:08] Diego). O esquema completo de colunas fica no FDD (modelo de dados), e a mecânica de leitura/despacho pelo worker é tratada no **ADR-002**.

Escopo desta decisão restringe-se ao **registro atômico** do evento. As decisões vizinhas ficam em ADRs próprios: leitura por worker/polling (ADR-002), retry/backoff/DLQ (ADR-003), assinatura HMAC (ADR-004), semântica *at-least-once* (ADR-005) e *snapshot* do payload na inserção (ADR-007).

---

## Alternativas Consideradas

| ID | Alternativa | Por que foi descartada | Origem |
|----|-------------|------------------------|--------|
| RFC-ALT-01 | **Disparo síncrono** do webhook dentro do `order.service` (`changeStatus`) | A transação de status já é pesada; um HTTP call no meio travaria a mudança de status de outros pedidos quando o cliente for lento. E, se o cliente estiver offline, exigiria dar *rollback* na mudança de status — inaceitável. | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | **Redis Streams / fila externa** | Exigiria subir mais infraestrutura (Redis Cluster) — *overengineering* para um time pequeno. A outbox no MySQL já existente resolve o mesmo problema sem nova stack. | [09:07] Larissa/Diego |

> A alternativa de **trigger de banco** para reatividade (RFC-ALT-03) também foi discutida, mas pertence à decisão de *como o worker lê* a outbox (ADR-002), não ao registro atômico do evento, e por isso não é avaliada aqui.

---

## Consequências

### Positivas

- **Consistência garantida (atomicidade).** Status e evento são confirmados ou revertidos juntos, na mesma transação; não há janela para "status mudou mas evento não saiu" ([09:06] Diego; PRD-NFR-05).
- **Sem nova infraestrutura.** Reaproveita MySQL + Prisma já em produção; nada de Redis Cluster para operar e monitorar — adequado ao tamanho do time [09:07] Diego.
- **Desacoplamento do HTTP da transação crítica.** A entrega de rede fica fora da transação de negócio; um cliente lento ou offline não trava a mudança de status de outros pedidos ([09:04] Bruno) — exatamente o defeito da alternativa síncrona.
- **Reuso do gancho existente.** A mudança de status já roda em `this.prisma.$transaction` (`src/modules/orders/order.service.ts`, L126-179); a outbox entra nessa mesma transação com alteração cirúrgica.

### Negativas / trade-offs

- **Entrega deixa de ser imediata.** Ao trocar o disparo síncrono pela outbox, aceita-se latência entre o *commit* e a entrega HTTP real, feita depois por um worker de forma assíncrona (mínimo de ~2s no pior caso — ver ADR-002). O trade-off é aceito porque, para os clientes, "tempo real" = abaixo de 10 segundos [09:02] Marcos.
- **Uma escrita a mais na transação quente.** A transação de status ganha o `INSERT` na `webhook_outbox`, aumentando levemente seu escopo. Como a inserção está dentro da transação, uma falha nela força *rollback* de toda a mudança de status ([09:40] Bruno, [09:41] Diego) — o registro do evento passa a ser acoplado à transação de negócio (efeito intencional da garantia de atomicidade).
- **Acúmulo de linhas entregues.** A tabela cresce com eventos já entregues; o arquivamento (~30 dias) fica **fora do escopo desta feature** [09:08] Diego (PRD-OOS-06).
- **Exige componentes complementares.** A outbox por si só não entrega nada: depende de um worker separado (ADR-002) e introduz semântica *at-least-once* delegada ao cliente via `X-Event-Id` (ADR-005). Estas são decisões separadas, referenciadas e não detalhadas aqui.

**Trade-off central, explícito:** troca-se a simplicidade e a imediatez de uma chamada HTTP síncrona por **consistência forte + resiliência**, ao custo de **latência assíncrona** e de **uma escrita adicional acoplada** à transação de mudança de status.

---

## Referências

**Reunião (TRANSCRICAO.md)**

- [09:00] Marcos — contexto de negócio: 3 clientes B2B pedindo notificação de mudança de status; core da feature (PRD-FR-07).
- [09:02] Marcos — "tempo real" = abaixo de 10s.
- [09:03] Larissa — questão arquitetural: síncrono vs. fila/outbox.
- [09:04] Bruno — transação de status já é pesada; não dá *rollback* por cliente offline (RFC-ALT-01).
- [09:06] Diego — proposta e nomeação do padrão Outbox; garantia commit/rollback conjunto (D1).
- [09:07] Larissa/Diego — descarte de Redis Streams / fila externa por *overengineering* (RFC-ALT-02).
- [09:08] Diego — índices (status, `created_at`); arquivamento de ~30 dias fora de escopo (PRD-OOS-06).
- [09:08] Larissa — decisão fechada: outbox em MySQL.
- [09:40] Bruno / [09:41] Diego — falha na inserção da outbox ⇒ *rollback*; função `publishWebhookEvent(tx, ...)` (PRD-NFR-05, FDD-INT-01).
- [09:51] Larissa — id da outbox = UUID (padrão do projeto).

**Código (fonte da verdade de integração)**

- `src/modules/orders/order.service.ts` — `changeStatus` (L126-179), já executa dentro de `this.prisma.$transaction`; ponto onde a inserção na outbox será acoplada.
- `prisma/schema.prisma` — datasource MySQL; ids `@default(uuid()) @db.Char(36)`, `@@map` e índices por convenção; base para o novo model `webhook_outbox`.

**ADRs relacionados (não duplicar altitude)**

- ADR-002 — Worker em processo separado com polling de 2s (leitura/despacho da outbox).
- ADR-003 — Retry com backoff exponencial e DLQ.
- ADR-005 — Entrega *at-least-once* com `X-Event-Id`.
- ADR-007 — *Snapshot* do payload na inserção da outbox.
