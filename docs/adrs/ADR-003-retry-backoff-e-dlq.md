# ADR-003 — Retry com backoff exponencial e Dead Letter Queue

| | |
|---|---|
| **Status** | Aceito |
| **Data** | 15 de julho de 2026 |
| **Autor** | Gabriele Rocha |
| **Decisão coberta** | D2 (fact-base §1) |
| **Decisores** | Diego (Eng. Sênior – Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno – Pedidos), Marcos (PM), Sofia (Eng. de Segurança) |

> Escopo desta ADR: **uma decisão** — a política de reação à falha de entrega de um webhook (quantas vezes retentar, com que espaçamento e o que fazer ao esgotar). A mecânica de leitura/gravação da outbox está na ADR-001, o worker que executa as tentativas está na ADR-002, e o contrato/modelo de dados detalhado está no FDD. Esta ADR não repete esses níveis.

---

## Status

Aceito. Decisão fechada por Larissa em [09:17], após Diego propor a progressão do backoff e Marcos aceitar o teto. Bruno havia sugerido 3 tentativas [09:16] (descartada) e endossou o pacote no fechamento geral da reunião [09:49].

---

## Contexto

O sistema entrega webhooks **outbound** (nós → cliente) para endpoints que estão **fora da nossa infraestrutura**, sob controle dos clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo [09:00] Marcos, [09:19] Sofia. A entrega, portanto, depende de um serviço de terceiro que pode estar indisponível ou lento no momento do envio.

A entrega é **assíncrona**: o worker (ADR-002) lê os eventos pendentes da outbox (ADR-001) e dispara o HTTP call. Uma tentativa de entrega é considerada **falha** quando a chamada não conclui — cliente offline ou erro de conexão [09:15] Diego — ou estoura o **timeout de 10 segundos** definido para o worker [09:42] Diego. (Tratar respostas com status não-2xx do endpoint como falha é convenção de implementação padrão, detalhada no FDD — não uma decisão tomada na reunião.) Diante disso, o sistema precisa de uma política explícita para o que fazer quando um envio falha; sem ela, o evento ou se perde ou fica pendurado indefinidamente.

Restrições e fatos que moldaram a decisão:

| ID | Fato / restrição | Origem |
|----|------------------|--------|
| CTX-1 | Cliente pode estar offline; já houve indisponibilidade **planejada de 2h** de um cliente | [09:16] Diego |
| CTX-2 | Retry indefinido deixa o evento "pendurado pra sempre" se o cliente sumiu de vez | [09:15] Diego |
| CTX-3 | Uma tentativa falha por cliente offline/erro de conexão ou timeout de 10s | [09:15][09:42] Diego |
| CTX-4 | Objetivo: tolerar indisponibilidade do cliente **sem perder evento** (PRD-OBJ-03) | [09:17] Diego |

Riscos endereçados: cliente offline por período longo perde eventos (PRD-RISK-01) [09:15]–[09:18]; cliente lento trava o processamento (PRD-RISK-05) [09:42].

---

## Decisão

Adotar **retry automático com backoff exponencial** e, ao esgotar as tentativas, mover o evento para uma **Dead Letter Queue (DLQ) persistida em tabela separada**, com **replay manual** via endpoint administrativo.

**1. Backoff exponencial com teto de 5 tentativas** [09:15][09:17] Diego, fechado por Larissa [09:17]:

| ID | Tentativa (após falha) | Espera até a próxima | Origem |
|----|------------------------|----------------------|--------|
| BO-1 | 1ª | 1 minuto | [09:17] Diego |
| BO-2 | 2ª | 5 minutos | [09:17] Diego |
| BO-3 | 3ª | 30 minutos | [09:17] Diego |
| BO-4 | 4ª | 2 horas | [09:17] Diego |
| BO-5 | 5ª (última) | 12 horas | [09:17] Diego |

Total: **~15 horas** entre a primeira falha e a última tentativa [09:17] Diego. Marcos aceitou o teto explicitamente: um cliente caído por 15h "já está com problema sério dele" [09:17].

**2. Dead Letter Queue em tabela separada** [09:18] Diego: esgotadas as 5 tentativas, o evento vira falha permanente e é movido para a tabela `webhook_dead_letter`, guardando **payload, motivo da falha e timestamp**. Manter a DLQ separada da outbox mantém a leitura da outbox principal limpa e dá evidência para debug e reprocessamento.

**3. Replay manual via endpoint admin** [09:18] Diego: reprocessamento é **manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O endpoint exige **role ADMIN** — reaproveitando `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` — e **registra quem fez o replay para auditoria** [09:36] Sofia, [09:36] Larissa.

> **Altitude:** a contagem de tentativas e o agendamento do próximo envio (campos de estado do evento, ex.: `attempts` / `next_attempt_at`) são detalhados no FDD e operados pelo worker da ADR-002. A duplicação que o replay pode gerar é tratada pela garantia at-least-once com `X-Event-Id` (ADR-005).

---

## Alternativas Consideradas

| ID | Alternativa | Por que foi descartada | Origem |
|----|-------------|------------------------|--------|
| ALT-A | **Retry indefinido** com backoff (RFC-ALT-04) | Deixa o evento pendurado para sempre quando o cliente sumiu de vez; sem teto não há critério para declarar falha permanente | [09:15] Diego |
| ALT-B | **3 tentativas** (RFC-ALT-04) | Sugerido por Bruno como opção mais agressiva; descartado por Diego: retentaria ~3x em cerca de 30 minutos e mataria o evento cedo demais, sem cobrir a indisponibilidade **planejada de 2h** já observada em cliente real | [09:16] Bruno / [09:16] Diego |
| ALT-C | **Marcar o evento como `failed` na própria outbox** (sem tabela DLQ separada) | Larissa levantou a opção; descartada em favor de tabela separada, que mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento | [09:17] Larissa / [09:18] Diego |

O teto de **5 tentativas** é o meio-termo escolhido entre ALT-A (nunca desiste) e ALT-B (desiste cedo demais): cobre uma janela de 12–24h de indisponibilidade sem manter eventos vivos indefinidamente [09:15] Diego.

---

## Consequências

### Positivas

- **Tolera indisponibilidade prolongada sem perder evento** (PRD-OBJ-03): a janela de ~15h absorve quedas reais, incluindo a indisponibilidade planejada de 2h já observada [09:16][09:17].
- **Outbox principal permanece limpa**: eventos mortos saem para a DLQ, então a varredura do worker nas linhas pendentes não carrega histórico de falhas permanentes [09:18].
- **Rastreabilidade para debug**: a DLQ guarda payload + motivo + timestamp, servindo de evidência para investigar por que um endpoint falhou [09:18].
- **Recuperação controlada e auditável**: o replay é uma ação deliberada de um ADMIN, registrada para auditoria, em vez de um reprocessamento automático que poderia bombardear um endpoint ainda instável [09:18][09:36].

### Negativas (trade-offs assumidos)

- **Latência de recuperação alta no pior caso**: um evento pode levar até ~15h para ser declarado morto, e um cliente que volta logo após a 4ª tentativa pode esperar até 12h pela 5ª. **Trade-off aceito** explicitamente: Marcos considerou que um cliente fora do ar por 15h já tem um problema grave próprio [09:17].
- **Reprocessamento é manual, não automático**: recuperar um item da DLQ exige um operador ADMIN agir [09:18]. Nesta fase **não há aviso automático** ao cliente sobre webhook com problema — o email de alerta ficou explicitamente para uma próxima fase, após medir o impacto [09:37] Larissa/Marcos.
- **Progressão fixa e global**: os intervalos e o teto de 5 são a política única decidida; ajustá-los exige mudança de configuração/código, não há personalização por endpoint (não discutida na reunião).
- **Timeout pode gerar reentrega**: como o timeout de 10s conta como falha [09:42], um cliente que processou o evento mas respondeu tarde receberá o mesmo evento de novo no retry. Isso é coerente com a garantia **at-least-once**, cuja deduplicação é delegada ao cliente via `X-Event-Id` (ADR-005) [09:24]–[09:25].

---

## Referências

**Reunião (TRANSCRICAO.md):**
- [09:15] Diego — backoff exponencial; teto de tentativas → DLQ; retry indefinido descartado; 5 cobre janela de 12–24h; cliente offline/erro de conexão conta como falha.
- [09:16] Bruno — sugestão de 3 tentativas; [09:16] Diego — 3 é pouco (indisponibilidade planejada de 2h já observada).
- [09:17] Diego — progressão 1m/5m/30m/2h/12h (~15h); [09:17] Marcos — aceita o teto; [09:17] Larissa — decisão fechada (5 tentativas, backoff definido); opção de marcar `failed` na própria outbox levantada aqui.
- [09:18] Diego — DLQ em tabela separada `webhook_dead_letter` (payload, motivo, timestamp); replay manual via `POST /admin/webhooks/dead-letter/:id/replay` recolocando na outbox como pendente.
- [09:36] Sofia / [09:36] Larissa — replay exige role ADMIN e loga quem executou (auditoria).
- [09:37] Larissa/Marcos — email de aviso ao cliente fora desta fase.
- [09:42] Diego — timeout de 10s do HTTP call conta como falha e dispara retry.

**Fact-base canônica:** §1 (D2), §7 (PRD-OOS-01), §9 (RFC-ALT-04), §10 (PRD-RISK-01, PRD-RISK-05), §6 (PRD-OBJ-03), §4 (PRD-FR-11), §5 (PRD-NFR-02).

**Código:**
- `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` reutilizado no endpoint de replay da DLQ (FDD-INT-05).

**ADRs relacionadas:** ADR-001 (Outbox no MySQL — origem/estado do evento), ADR-002 (Worker em processo separado com polling de 2s — executa as tentativas e agenda o backoff), ADR-005 (At-least-once com `X-Event-Id` — deduplicação do lado do cliente para reentregas).
