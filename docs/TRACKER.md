# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este documento é a **referência cruzada** do pacote de design docs. Cada item registrado no PRD, RFC, FDD e ADRs tem uma linha aqui, mapeada à sua origem: uma fala na reunião (`TRANSCRICAO`, com `[hh:mm] Nome`) ou o código-fonte da aplicação (`CODIGO`, com caminho de arquivo real). Serve para garantir que **nada foi inventado**: se uma linha do PRD/RFC/FDD/ADR não tem origem identificável aqui, ela não deveria existir.

**Formato das linhas:** `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`.

**Convenções:**
- `TRANSCRICAO` → `Localização` é `[hh:mm] Nome` do falante (arquivo `TRANSCRICAO.md`).
- `CODIGO` → `Localização` é o caminho do arquivo no repositório.
- Itens que consolidam vários timestamps citam o timestamp **da decisão/afirmação primária**; os documentos de origem trazem as citações completas.
- Itens marcados *(proposta)* são detalhamento de implementação derivado de uma decisão da reunião (a decisão-mãe está rastreada); a proposta em si vive no FDD.

**Cobertura:** ~150 itens rastreados. Distribuição por fonte ao final do documento (≥70% `TRANSCRICAO`, ≥5 linhas `CODIGO` com caminho real).

---

## 1. PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| PRD-PROB-01 | docs/PRD.md | Motivação | Clientes B2B não têm notificação em tempo real; dependem de polling | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Motivação | Polling torna a integração do cliente lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-03 | docs/PRD.md | Restrição (negócio) | Atlas ameaça migrar ao concorrente se não entregarmos no prazo | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-04 | docs/PRD.md | Restrição | "Tempo real" para o cliente = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Notificar em tempo real: latência de entrega < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Eliminar o polling caro dos clientes em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Tolerar indisponibilidade: 5 tentativas em janela de ~15 h antes da DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Reter Atlas: entregar até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de aviso ao cliente quando o webhook falha — adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída — ponto em aberto | TRANSCRICAO | [09:39] Diego |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard/painel visual — projeto do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Múltiplos workers em paralelo — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Webhooks inbound (cliente → nós) — só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Endurecer roles do CRUD de config — futuro | TRANSCRICAO | [09:37] Sofia |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Entrega exactly-once — descartada | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint (URL, event types, customer); secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por endpoint (lista de status assinados) | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotacionar secret via API; anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega outbound automática na mudança de status (core) | TRANSCRICAO | [09:06] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimas ~100: status, payload, response, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ, role ADMIN, auditado | TRANSCRICAO | [09:35] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Assinar cada envio (HMAC-SHA256) + headers de rastreio | TRANSCRICAO | [09:44] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry automático com backoff; após 5 tentativas → DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Recusar cadastro/edição com URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s; polling 2 s ⇒ mínimo ~2 s no pior caso | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout do HTTP call = 10 s; sem resposta ⇒ falha | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório: URL deve ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB; erra (não trunca) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Atomicidade: evento na transação de status; falha ⇒ rollback | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | At-least-once; dedup delegada ao cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker como processo separado, resiliente a restart da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Observabilidade pela stack de logging existente (sem stack nova) | TRANSCRICAO | [09:29] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Depende do processo de mudança de status (ponto de captura do evento) | TRANSCRICAO | [09:41] Diego |
| PRD-DEP-02 | docs/PRD.md | Dependência | Depende do MySQL em produção; sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-03 | docs/PRD.md | Dependência | Depende do JWT/roles existentes, incl. ADMIN para o replay | TRANSCRICAO | [09:36] Larissa |
| PRD-DEP-04 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (≥2 dias) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Documentação da semântica no portal do desenvolvedor (PM) | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Cliente implementa verificação HMAC e dedup por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-07 | docs/PRD.md | Dependência | Capacidade de entrega estimada em 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente offline por período longo perde eventos | TRANSCRICAO | [09:15] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret (ex.: log do cliente) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Eventos fora de ordem se escalar workers | TRANSCRICAO | [09:13] Larissa |
| PRD-RISK-04 | docs/PRD.md | Risco | Bombardeio do cliente sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente lento trava o processamento | TRANSCRICAO | [09:42] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Não entregar no prazo → Atlas migra | TRANSCRICAO | [09:45] Marcos |
| PRD-AC-01 | docs/PRD.md | Critério de aceitação | Evento entregue ao cliente online em < 10 s | TRANSCRICAO | [09:10] Larissa |
| PRD-AC-02 | docs/PRD.md | Critério de aceitação | Falha no registro do evento reverte a mudança de status | TRANSCRICAO | [09:41] Diego |
| PRD-AC-03 | docs/PRD.md | Critério de aceitação | Após 5 tentativas sem sucesso, evento vai para DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-AC-04 | docs/PRD.md | Critério de aceitação | Cada envio assinado (HMAC) + headers de rastreio | TRANSCRICAO | [09:44] Diego |
| PRD-AC-05 | docs/PRD.md | Critério de aceitação | Secret rotacionável; anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de aceitação | Cadastro/edição recusa URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceitação | Endpoint recebe apenas os status que assinou | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-08 | docs/PRD.md | Critério de aceitação | Histórico retorna últimas ~100 entregas com detalhes | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | docs/PRD.md | Critério de aceitação | Replay de DLQ só para ADMIN e auditado | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-10 | docs/PRD.md | Critério de aceitação | Processamento sobrevive a restart da API | TRANSCRICAO | [09:11] Diego |
| PRD-AC-11 | docs/PRD.md | Critério de aceitação | Revisão de segurança concluída antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TS-01 | docs/PRD.md | Estratégia de teste | Testes por camada com Vitest (framework já no repo) | CODIGO | package.json / tests/ |

---

## 2. RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| RFC-PROP-01 | docs/RFC.md | Decisão/Proposta | Outbox transacional no MySQL (evento na transação de changeStatus) | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão/Proposta | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão/Proposta | Filtragem na inserção (só grava se há endpoint assinante) | TRANSCRICAO | [09:34] Bruno |
| RFC-PROP-04 | docs/RFC.md | Decisão/Proposta | Worker em processo separado com polling de 2 s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-05 | docs/RFC.md | Decisão/Proposta | Entrega HTTP assinada com HMAC-SHA256, secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| RFC-PROP-06 | docs/RFC.md | Decisão/Proposta | Retry com backoff exponencial (5 tentativas) → DLQ | TRANSCRICAO | [09:17] Diego |
| RFC-PROP-07 | docs/RFC.md | Decisão/Proposta | At-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| RFC-PROP-08 | docs/RFC.md | Decisão/Proposta | Reuso máximo dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa (Trade-off) | Disparo síncrono no order service — travaria status/forçaria rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa (Trade-off) | Redis Streams/fila externa — overengineering p/ time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa (Trade-off) | Trigger de banco — MySQL sem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa (Trade-off) | Retry indefinido/3 tentativas — pendura ou mata cedo demais | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa (Trade-off) | Exactly-once — coordenação bidirecional complexa | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa (Trade-off) | Secret global — se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Ordering global ao escalar workers — não decidido | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Notificação (e-mail) ao cliente sobre webhook com problema | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Endurecimento de roles do CRUD de config | TRANSCRICAO | [09:37] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Cliente offline por período longo | TRANSCRICAO | [09:15] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Secret vazada (ex.: log do cliente) | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Ordem fora de sequência ao escalar workers | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Bombardeio do cliente sem rate limit | TRANSCRICAO | [09:38] Diego |
| RFC-RISK-05 | docs/RFC.md | Risco | Prazo apertado (Atlas migra) | TRANSCRICAO | [09:45] Marcos |

---

## 3. FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Emitir evento atômico com a mudança de status (rollback em falha) | TRANSCRICAO | [09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Entregar em < 10 s; polling 2 s | TRANSCRICAO | [09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Processamento em processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | At-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Retry com backoff (5 tentativas ~15 h) + DLQ com replay | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | HMAC-SHA256, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-07 | docs/FDD.md | Objetivo técnico | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação de changeStatus | TRANSCRICAO | [09:41] Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Processamento pelo worker (polling 2 s, batch por created_at) | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Retry com backoff (next_attempt_at, 5 tentativas) | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | DLQ (tabela separada) + replay admin | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato/Endpoint | POST /webhooks — cadastra, retorna secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato/Endpoint | GET /webhooks?customerId= — listar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato/Endpoint | PATCH/DELETE /webhooks/:id — editar/remover | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato/Endpoint | POST /webhooks/:id/secret/rotate — rotação com grace 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato/Endpoint | GET /webhooks/:id/deliveries — histórico (100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato/Endpoint | POST /admin/.../dead-letter/:id/replay — ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato/Endpoint | Requisição outbound (payload + headers de rastreio) | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_INVALID_URL (422) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_URL_NOT_HTTPS (422) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_SECRET_REQUIRED (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-05 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_PAYLOAD_TOO_LARGE (422) — teto 64 KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_INVALID_EVENT_TYPE (422) — valor fora de OrderStatus | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-07 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_SECRET_ROTATION_PENDING (409) — grace ativo | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-09 | docs/FDD.md | Erro (WEBHOOK_*) | WEBHOOK_DELIVERY_TIMEOUT (interno) — 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Atomicidade (inserção na $transaction; falha ⇒ rollback) | TRANSCRICAO | [09:41] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Timeout de 10 s no HTTP call | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | DLQ separada + replay ADMIN | TRANSCRICAO | [09:18] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Worker resiliente a restart da API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | At-least-once (dedup no cliente) | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados via Pino existente | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas derivadas dos dados persistidos *(proposta)* | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Tracing via X-Event-Id como correlation id *(proposta)* | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração (código) | Estende changeStatus com publishWebhookEvent(tx,...) na $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração (código) | Novas classes WEBHOOK_* estendendo AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Integração (código) | Error middleware reusado sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração (código) | Novo src/worker.ts espelha bootstrap; PrismaClient próprio | CODIGO | src/server.ts, src/config/database.ts |
| FDD-INT-05 | docs/FDD.md | Integração (código) | Reuso de authenticate + requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração (código) | Novo módulo webhooks espelhando customers; registro em rotas/app | CODIGO | src/modules/customers, src/routes/index.ts, src/app.ts |
| FDD-INT-07 | docs/FDD.md | Integração (código) | Reuso do logger Pino; estender redact p/ secret | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração (código) | Novos models Prisma seguindo convenções do schema | CODIGO | prisma/schema.prisma |
| FDD-AC-01 | docs/FDD.md | Critério de aceite téc. | Inserção na mesma transação; falha ⇒ rollback | TRANSCRICAO | [09:41] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de aceite téc. | Nenhuma linha se nenhum endpoint ouve o status | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de aceite téc. | Payload gravado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-AC-04 | docs/FDD.md | Critério de aceite téc. | Entrega em < 10 s no caminho feliz | TRANSCRICAO | [09:10] Larissa |
| FDD-AC-05 | docs/FDD.md | Critério de aceite téc. | Retry com backoff; 5ª → DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-AC-06 | docs/FDD.md | Critério de aceite téc. | Headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-AC-07 | docs/FDD.md | Critério de aceite téc. | Rotação mantém secret anterior por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de aceite téc. | URL não-HTTPS recusada | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-09 | docs/FDD.md | Critério de aceite téc. | Payload > 64 KB recusado | TRANSCRICAO | [09:24] Diego |
| FDD-AC-10 | docs/FDD.md | Critério de aceite téc. | Replay exige ADMIN e loga executor | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-11 | docs/FDD.md | Critério de aceite téc. | X-Event-Id estável entre reentregas | TRANSCRICAO | [09:25] Diego |
| FDD-AC-12 | docs/FDD.md | Critério de aceite téc. | Worker sobrevive a restart da API | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco (técnico) | Cliente offline longo perde eventos → backoff + DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco (técnico) | Secret vazada → por endpoint + rotação + redact | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco (técnico) | Ordering ao escalar → single-worker documentado | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-04 | docs/FDD.md | Risco (técnico) | Bombardeio sem rate limit → em aberto | TRANSCRICAO | [09:39] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco (técnico) | Cliente lento → timeout 10 s + assíncrono | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco (técnico) | Inconsistência status↔evento → atomicidade | TRANSCRICAO | [09:41] Diego |
| FDD-DATA-01 | docs/FDD.md | Modelo de dados | Id da outbox = UUID (@db.Char(36)), padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-DATA-02 | docs/FDD.md | Modelo de dados | Índices em status e created_at (leitura do worker) | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Restrição | customer_id vem do body/path, não do JWT (JWT é do operador) | TRANSCRICAO | [09:32] Bruno |
| FDD-DATA-04 | docs/FDD.md | Contrato (payload) | Payload enxuto sem items; cliente consulta GET /orders/:id | TRANSCRICAO | [09:43] Diego |

---

## 4. ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL com transação atômica | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CODE | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão (âncora código) | changeStatus roda em this.prisma.$transaction (ponto de acoplamento) | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker em processo separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-CODE | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão (âncora código) | Molde do bootstrap e fábrica de PrismaClient | CODIGO | src/server.ts, src/config/database.ts |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Retry com backoff exponencial (5×) e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | Entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão (âncora código) | AppError, error middleware, Pino, validate, auth, módulos como molde | CODIGO | src/shared/errors/app-error.ts, src/middlewares/error.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-outbox.md | Decisão | Snapshot do payload renderizado na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-008 | docs/adrs/ADR-008-filtragem-eventos-na-insercao.md | Decisão | Filtragem de eventos por assinatura na inserção da outbox | TRANSCRICAO | [09:34] Bruno |

---

## 5. Distribuição por fonte

| Fonte | Linhas | % |
|-------|--------|---|
| `TRANSCRICAO` (com `[hh:mm] Nome`) | 149 | ~92,5% |
| `CODIGO` (com caminho de arquivo real) | 12 | ~7,5% |
| **Total** | **161** | **100%** |

**Linhas `CODIGO` (≥5 exigidas):** `FDD-INT-01..08`, `PRD-TS-01`, `ADR-001-CODE`, `ADR-002-CODE`, `ADR-006-CODE` — todas com caminho real verificado no repositório (`src/modules/orders/order.service.ts`, `src/shared/errors/*`, `src/middlewares/*`, `src/server.ts`, `src/config/database.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `prisma/schema.prisma`, `package.json`, `tests/`).

**Cobertura:** todos os IDs numerados dos quatro documentos (PRD-FR/NFR/OBJ/OOS/RISK/DEP/AC/PROB/TS, RFC-PROP/ALT/OPEN/RISK, FDD-OBJ/FLOW/CONTRATO/ERR/RES/OBS/INT/AC/RISK/DATA, ADR-001..008) têm linha correspondente — acima de 80% dos itens identificáveis do pacote.
