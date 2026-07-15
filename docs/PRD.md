# PRD — Sistema de Webhooks de Notificação de Pedidos

| | |
|---|---|
| **Documento** | Product Requirements Document (PRD) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Produto** | Order Management System (OMS) — Node.js + TypeScript + Express + Prisma + MySQL (em produção) |
| **Autora** | Gabriele Rocha |
| **Data de elaboração** | 15 de julho de 2026 |
| **Revisores / decisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. de Segurança) |
| **Fonte** | Reunião técnica de definição (transcrição) + fact-base canônica |
| **Altitude** | Produto/negócio ("por quê" e "o quê"). Detalhes de implementação (contratos, modelo de dados, matriz de erros, fluxos) vivem no FDD e nos ADRs. |

> Rastreabilidade: cada requisito, objetivo, restrição, decisão e risco deste documento tem origem citada — `[hh:mm]` referencia a fala na reunião de definição da feature. Decisões arquiteturais estão resumidas aqui e detalhadas nos ADRs referenciados por nome.

---

## 1. Resumo e contexto da feature

O OMS hoje só expõe o estado dos pedidos por consulta ativa: os clientes B2B integrados precisam bater periodicamente em nossa API de pedidos para descobrir se algo mudou. Esta feature inverte esse modelo, introduzindo **notificações outbound (push)**: sempre que o status de um pedido muda na plataforma, o sistema envia automaticamente uma notificação HTTP assinada para os endpoints que o cliente cadastrou, em tempo quase real.

O escopo cobre o ciclo completo do lado do provedor: cadastro e gestão dos endpoints de webhook por cliente, geração e rotação de segredos de assinatura, captura confiável do evento no momento da mudança de status, entrega assíncrona com tentativas automáticas, fila de eventos não entregues (dead-letter) com reprocessamento manual, e histórico de entregas consultável. A comunicação é exclusivamente **outbound (nós → cliente)**; o cliente recebe, não envia [09:02] Marcos, [09:03] Sofia.

A solução foi desenhada para reaproveitar ao máximo a stack e os padrões já em produção (mesmo MySQL, mesmo logger, mesmos padrões de módulo e de erro), evitando infraestrutura nova [09:07] Diego, [09:30] Larissa. Os detalhes de arquitetura estão nos ADRs (seção 8); os contratos técnicos, modelo de dados e fluxos ficam no FDD.

---

## 2. Problema e motivação

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — pediram formalmente para serem notificados em tempo real quando o status de seus pedidos muda na plataforma [09:00] Marcos.

Hoje esses clientes fazem *polling*: batem repetidamente na API de listagem de pedidos de tempos em tempos para descobrir mudanças. Esse modelo deixa a integração deles **lenta e cara**, e ainda os obriga a "ficar atualizando manualmente" para não perder uma mudança [09:00][09:02] Marcos.

O problema tem urgência comercial concreta: a **Atlas Comercial sinalizou que pode migrar para o concorrente** caso a notificação em tempo real não seja entregue até o fim do trimestre [09:00] Marcos. Portanto, além do ganho de produto (integração mais eficiente para o cliente), a feature tem impacto direto em **retenção de receita B2B**.

| ID | Motivação | Origem |
|----|-----------|--------|
| PRD-PROB-01 | Clientes B2B não têm notificação em tempo real de mudança de status; dependem de polling | [09:00] Marcos |
| PRD-PROB-02 | Polling torna a integração do cliente lenta e cara | [09:00] Marcos |
| PRD-PROB-03 | Risco de churn: Atlas ameaça migrar para o concorrente se não entregarmos no prazo | [09:00] Marcos |
| PRD-PROB-04 | "Tempo real" para os clientes = qualquer coisa abaixo de 10 segundos | [09:02] Marcos |

---

## 3. Público-alvo e cenários de uso

**Público-alvo:** clientes B2B integrados ao OMS via API, representados no sistema por usuários autenticados. Os três demandantes iniciais são Atlas Comercial, MaxDistribuição e Nova Cargo [09:00] Marcos.

### Cenários de uso

- **Atlas Comercial — retenção sob prazo.** Cliente estratégico e em risco de churn. Precisa substituir o polling caro por recebimento push antes do fim de novembro. É o cenário que dita o prazo da entrega [09:00][09:45] Marcos.

- **MaxDistribuição — integração eficiente.** Deixa de consultar a API repetidamente e passa a reagir apenas às mudanças de status que lhe interessam, assinando somente os status relevantes ao seu fluxo (filtro por evento) [09:00] Marcos, [09:33] Marcos.

- **Nova Cargo — resiliência e múltiplos endpoints.** Cliente que precisa tolerar indisponibilidade temporária do próprio endpoint sem perder eventos (tentativas automáticas) e, tendo mais de um cadastro de webhook, saber a qual endpoint cada notificação se refere [09:00] Marcos, [09:15] Diego, [09:44] Sofia.

Observação de escopo: os clientes **nunca pediram garantia de ordenação global** de eventos; querem apenas saber se cada pedido mudou [09:14] Marcos. Isso delimita expectativas de produto (ver riscos, PRD-RISK-03).

---

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica / meta quantitativa | Origem |
|----|----------|------------------------------|--------|
| PRD-OBJ-01 | Notificar clientes em "tempo real" | Latência de entrega **< 10 segundos** para eventos com cliente online (ciclo de leitura do processamento a cada 2 s ⇒ ~2 s no pior caso) | [09:02] Marcos, [09:10] Larissa |
| PRD-OBJ-02 | Eliminar o polling caro do cliente | Substituir a consulta repetida à API de pedidos por entrega push (redução das chamadas de leitura dos clientes) | [09:00] Marcos |
| PRD-OBJ-03 | Tolerar indisponibilidade do cliente sem perder evento | **5 tentativas** automáticas ao longo de uma janela de **~15 horas** antes de mover o evento para a fila de não entregues (DLQ) | [09:15][09:17] Diego |
| PRD-OBJ-04 | Reter os clientes B2B em risco (Atlas) | Entregar a feature até **o fim de novembro** / fim do trimestre | [09:00][09:45] Marcos |

O objetivo primário e mensurável é **PRD-OBJ-01**: entrega abaixo de 10 segundos. A meta de tolerância a falhas (PRD-OBJ-03) tem meta quantitativa explícita — 5 tentativas na janela de ~15 h, com a progressão de backoff decidida na reunião e detalhada no ADR de retry.

---

## 5. Escopo

### 5.1 Incluso no escopo

- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente [09:31][09:33].
- Filtro de eventos por endpoint: cada cadastro escolhe quais status de pedido quer receber [09:33] Marcos.
- Geração de segredo de assinatura por endpoint e rotação de segredo com período de convivência ("grace period") de 24 h [09:21] Sofia.
- Entrega outbound automática do evento quando o status do pedido muda [09:00] Marcos, [09:06] Diego.
- Assinatura de cada envio (HMAC-SHA256) e cabeçalhos de rastreio [09:20] Sofia, [09:44] Diego/Sofia.
- Tentativas automáticas com backoff em falha de entrega e fila de não entregues (DLQ) com reprocessamento manual restrito a perfil ADMIN [09:15]–[09:18] Diego, [09:36] Sofia/Larissa.
- Histórico de entregas consultável (últimas entregas por endpoint) [09:34] Marcos.
- Validação de que a URL do endpoint é HTTPS [09:23] Sofia.

### 5.2 Fora de escopo

A reunião **descartou ou adiou explicitamente** os itens abaixo. Nenhum deles pode ser tratado como requisito desta fase.

| ID | Item fora de escopo | Situação | Origem |
|----|---------------------|----------|--------|
| PRD-OOS-01 | **E-mail de aviso ao cliente quando o webhook falha** (ex.: após falhas seguidas) | **Adiado** — próxima fase, depois de medir impacto | [09:37] Larissa, [09:38] Marcos |
| PRD-OOS-02 | **Rate limiting de saída** (limitar o volume de chamadas quando muitos pedidos mudam ao mesmo tempo) | **Ponto em aberto** — "observar e decidir depois" | [09:38][09:39] Diego/Larissa |
| PRD-OOS-03 | **Dashboard / painel visual para o cliente** | **Fora de escopo** — apenas endpoints; painel é projeto do time de frontend | [09:39] Marcos, [09:40] Larissa |
| PRD-OOS-04 | Escalar para múltiplos workers em paralelo (partição por pedido, lock pessimista) | Futuro — "problema do futuro" | [09:13] Diego |
| PRD-OOS-05 | Webhooks inbound (cliente → nós) | Fora de escopo — só outbound | [09:02] Marcos/Sofia |
| PRD-OOS-06 | Arquivamento de eventos já entregues (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| PRD-OOS-07 | Endurecer os perfis de acesso do CRUD de configuração | Futuro — "mais pra frente a gente pode endurecer" | [09:37] Sofia |
| PRD-OOS-08 | Entrega exactly-once | Descartado — coordenação complexa; at-least-once resolve | [09:25] Diego |

Os três primeiros (**e-mail de aviso, rate limiting de saída e dashboard visual**) foram os cortes explicitamente confirmados no fechamento da reunião [09:48] Larissa.

---

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|----|-----------|--------|
| PRD-FR-01 | Cadastrar endpoint de webhook: URL, lista de status a receber e identificação do cliente; o segredo de assinatura é gerado pela plataforma e devolvido na criação. O identificador do cliente é informado na requisição, **não** derivado do JWT (o JWT é do operador) | [09:31][09:32] Marcos, [09:32] Bruno/Larissa |
| PRD-FR-02 | Editar um endpoint de webhook existente | [09:33] Bruno |
| PRD-FR-03 | Remover um endpoint de webhook | [09:33] Bruno |
| PRD-FR-04 | Listar os webhooks de um cliente | [09:33] Bruno |
| PRD-FR-05 | Permitir que cada endpoint assine apenas os status que deseja receber (filtro de eventos por endpoint); só os status assinados geram evento | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-06 | Rotacionar o segredo de assinatura de um endpoint via API; o segredo anterior permanece válido por 24 h (grace period) para migração do cliente | [09:21] Sofia |
| PRD-FR-07 | Entregar o evento outbound automaticamente quando o status de um pedido muda (funcionalidade central) | [09:00] Marcos, [09:06] Diego |
| PRD-FR-08 | Disponibilizar histórico de entregas por endpoint (últimas ~100 entregas: sucesso/falha, conteúdo enviado, resposta e tempo de resposta) | [09:34] Marcos |
| PRD-FR-09 | Reprocessar manualmente um evento da fila de não entregues (DLQ); restrito a perfil **ADMIN** e auditado (registra quem executou) | [09:35] Diego, [09:36] Sofia/Larissa |
| PRD-FR-10 | Assinar cada envio com HMAC-SHA256 e incluir cabeçalhos de rastreio (identificador do evento, assinatura, timestamp do envio e identificador do endpoint) | [09:20] Sofia, [09:44] Diego/Sofia |
| PRD-FR-11 | Retentar automaticamente com backoff em caso de falha de entrega; após 5 tentativas, mover o evento para a DLQ | [09:15]–[09:18] Diego |
| PRD-FR-12 | Recusar o cadastro/edição de endpoints cuja URL não seja HTTPS | [09:23] Sofia |

---

## 7. Requisitos não funcionais

| ID | Requisito não funcional | Origem |
|----|-------------------------|--------|
| PRD-NFR-01 | Latência de entrega inferior a 10 s para clientes online; ciclo de processamento a cada 2 s ⇒ mínimo de ~2 s no pior caso | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | Tempo limite (timeout) da chamada de entrega ao cliente = 10 s; ausência de resposta é tratada como falha e gera nova tentativa | [09:42] Diego |
| PRD-NFR-03 | TLS obrigatório: a URL do endpoint deve ser HTTPS; HTTP é recusado | [09:23] Sofia |
| PRD-NFR-04 | Limite de tamanho de conteúdo do evento de 64 KB; ultrapassado o limite, o envio falha (não trunca) | [09:23] Sofia, [09:24] Diego/Larissa |
| PRD-NFR-05 | Atomicidade: o registro do evento ocorre dentro da mesma transação da mudança de status; se o registro falhar, a mudança de status sofre rollback (não há status alterado sem evento) | [09:40] Bruno, [09:41] Diego |
| PRD-NFR-06 | Garantia de entrega at-least-once; não há garantia de ausência de duplicatas — a idempotência é delegada ao cliente pelo identificador do evento | [09:24]–[09:26] Diego |
| PRD-NFR-07 | O processamento de entregas roda como processo separado, resiliente a reinício da API | [09:11] Diego |
| PRD-NFR-08 | Observabilidade pela stack de logging já existente no projeto, sem introduzir stack nova | [09:29] Bruno |

---

## 8. Decisões e trade-offs principais

As seis decisões arquiteturais centrais estão registradas em ADRs próprios. O resumo abaixo aponta o "o quê" e o trade-off de negócio; o detalhamento (contexto, alternativas e consequências) vive em cada ADR — **não** é repetido aqui.

| Decisão | Trade-off resumido | ADR |
|---------|--------------------|-----|
| Captura do evento na mesma transação da mudança de status (padrão Outbox), sem infraestrutura nova | Garante consistência (evento existe se, e somente se, a mudança de status foi confirmada) ao custo de entrega assíncrona, em vez de disparo síncrono | [./adrs/ADR-001-outbox-no-mysql.md](./adrs/ADR-001-outbox-no-mysql.md) |
| Processamento por processo separado, em ciclo curto de leitura (2 s) | Simplicidade e resiliência a reinício da API, aceitando latência mínima de ~2 s no pior caso | [./adrs/ADR-002-worker-processo-separado-polling.md](./adrs/ADR-002-worker-processo-separado-polling.md) |
| Tentativas automáticas com backoff e fila de não entregues (DLQ) com replay manual | Tolera indisponibilidade longa do cliente (5 tentativas, ~15 h) sem pendurar eventos para sempre | [./adrs/ADR-003-retry-backoff-e-dlq.md](./adrs/ADR-003-retry-backoff-e-dlq.md) |
| Assinatura HMAC-SHA256 com segredo por endpoint e rotação com grace de 24 h | Cliente valida origem e integridade; blast radius de vazamento fica contido a um endpoint | [./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Entrega at-least-once com identificador de evento para deduplicação no cliente | Muito mais simples que exactly-once; transfere a deduplicação para o cliente (padrão de mercado) | [./adrs/ADR-005-at-least-once-x-event-id.md](./adrs/ADR-005-at-least-once-x-event-id.md) |
| Reuso máximo dos padrões existentes (módulos, tratamento de erro, logging, validação) | Menor custo e menor risco de manutenção, alinhado ao tamanho do time | [./adrs/ADR-006-reuso-padroes-existentes.md](./adrs/ADR-006-reuso-padroes-existentes.md) |

Duas decisões secundárias também foram fechadas na reunião e têm ADR próprio: o **snapshot do conteúdo do evento no momento da captura** ([./adrs/ADR-007-snapshot-payload-na-outbox.md](./adrs/ADR-007-snapshot-payload-na-outbox.md)) [09:52] Larissa/Diego/Bruno e a **filtragem de eventos no momento da captura**, e não no envio ([./adrs/ADR-008-filtragem-eventos-na-insercao.md](./adrs/ADR-008-filtragem-eventos-na-insercao.md)) [09:34] Bruno/Diego.

---

## 9. Dependências

| ID | Dependência | Origem |
|----|-------------|--------|
| PRD-DEP-01 | Depende do processo existente de mudança de status de pedidos, ponto onde o evento é capturado | [09:40] Bruno, [09:41] Diego |
| PRD-DEP-02 | Depende da infraestrutura de banco (MySQL) já em produção; nenhuma infraestrutura nova é provisionada (ex.: sem fila externa/Redis) | [09:07] Larissa/Diego |
| PRD-DEP-03 | Depende do mecanismo de autenticação/autorização existente (JWT e perfis), incluindo o perfil ADMIN para o replay de DLQ | [09:32] Larissa, [09:36] Larissa |
| PRD-DEP-04 | Depende de revisão de segurança pela Sofia (mínimo 2 dias úteis, com foco em HMAC e geração de segredo) antes do deploy | [09:46] Sofia |
| PRD-DEP-05 | Depende de documentação da integração e da semântica at-least-once no portal do desenvolvedor, sob responsabilidade do PM | [09:26][09:40] Marcos |
| PRD-DEP-06 | Depende de o cliente implementar a verificação da assinatura HMAC e a deduplicação pelo identificador do evento do lado dele | [09:20] Sofia, [09:25] Diego |
| PRD-DEP-07 | Depende da capacidade de entrega estimada em 3 sprints (revisão de segurança incluída) | [09:46] Larissa |

---

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
|----|-------|:-------------:|:-------:|-----------|--------|
| PRD-RISK-01 | Cliente offline por período longo perde eventos | Média | Alto | Tentativas com backoff (5 tentativas, ~15 h) + DLQ + reprocessamento manual | [09:15]–[09:18] Diego |
| PRD-RISK-02 | Vazamento de segredo (ex.: em log do próprio cliente) | Média | Alto | Segredo por endpoint (contém o blast radius) + rotação com grace de 24 h | [09:21][09:22] Sofia/Diego |
| PRD-RISK-03 | Eventos fora de ordem caso venha a escalar para múltiplos workers | Baixa | Médio | Manter processamento single-worker; limitação documentada; clientes não pedem ordenação global | [09:12]–[09:14] |
| PRD-RISK-04 | Bombardeio do cliente (muitos eventos por minuto) sem rate limiting | Média | Médio | Ponto em aberto: observar e implementar rate limiting de saída se virar problema (fora do escopo desta fase) | [09:38][09:39] Diego |
| PRD-RISK-05 | Cliente lento trava o processamento das entregas | Baixa | Médio | Timeout de 10 s → falha → nova tentativa; processamento assíncrono, fora da transação de status | [09:42] Diego |
| PRD-RISK-06 | Não entregar no prazo → Atlas migra para o concorrente | Média | Alto | 3 sprints planejadas + revisão de segurança agendada no fim | [09:00][09:45][09:46] |

---

## 11. Critérios de aceitação

| ID | Critério | Referência |
|----|----------|------------|
| PRD-AC-01 | Ao mudar o status de um pedido, o evento correspondente é entregue ao cliente online em menos de 10 s | PRD-OBJ-01, PRD-NFR-01 |
| PRD-AC-02 | Se o registro do evento falhar, a mudança de status é revertida; nunca há status alterado sem o evento correspondente registrado (e vice-versa) | PRD-NFR-05 |
| PRD-AC-03 | Após 5 tentativas de entrega sem sucesso, o evento é movido para a DLQ | PRD-FR-11, PRD-OBJ-03 |
| PRD-AC-04 | Cada envio é assinado com HMAC-SHA256 e acompanha os cabeçalhos de rastreio (identificador do evento, assinatura, timestamp e identificador do endpoint) | PRD-FR-10 |
| PRD-AC-05 | O segredo de um endpoint pode ser rotacionado, e o segredo anterior permanece válido por 24 h após a rotação | PRD-FR-06 |
| PRD-AC-06 | O cadastro/edição de endpoints recusa URLs não HTTPS | PRD-FR-12, PRD-NFR-03 |
| PRD-AC-07 | Um endpoint recebe apenas os status que assinou; status não assinados não geram evento para ele | PRD-FR-05 |
| PRD-AC-08 | O histórico de entregas de um endpoint retorna as últimas ~100 entregas com status, conteúdo, resposta e tempo de resposta | PRD-FR-08 |
| PRD-AC-09 | O reprocessamento de um item da DLQ só é permitido a perfil ADMIN e é auditado (registra quem executou) | PRD-FR-09 |
| PRD-AC-10 | O processamento de entregas continua funcionando após reinício da API (processo separado) | PRD-NFR-07 |
| PRD-AC-11 | A revisão de segurança da Sofia é concluída antes do deploy | PRD-DEP-04 |

---

## 12. Estratégia de testes e validação

A validação usa o **Vitest**, framework de testes já configurado no repositório (`package.json`, pasta `tests/`), mantendo o padrão de testes por módulo já existente — sem introduzir ferramental novo, coerente com a decisão de **reuso máximo dos padrões do projeto** [09:30] Larissa.

Camadas de cobertura previstas:

- **Unitária** — regras isoladas: captura do evento, cálculo do backoff das tentativas e geração/verificação da assinatura HMAC.
- **Integração** — comportamentos que cruzam limites: atomicidade entre mudança de status e registro do evento, e o ciclo de processamento até a entrega e o histórico.
- **Ponta a ponta (e2e)** — jornadas de API: CRUD de configuração de webhook, rotação de segredo e o reprocessamento de DLQ restrito a perfil ADMIN.

Além dos testes automatizados, há um **gate de validação de segurança**: a Sofia revisa o código de segurança (com foco em HMAC e geração de segredo) por, no mínimo, 2 dias úteis antes do deploy [09:46] Sofia. Essa revisão é condição de release (ver PRD-AC-11 e PRD-DEP-04). O detalhamento dos casos de teste por camada e por endpoint pertence ao FDD.
