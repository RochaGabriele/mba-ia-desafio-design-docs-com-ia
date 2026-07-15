# ADR-004 — Assinatura HMAC-SHA256 com secret única por endpoint e rotação

| Campo | Valor |
|-------|-------|
| **Status** | Aceito |
| **Data** | 15 de julho de 2026 |
| **Autor** | Gabriele Rocha |
| **Decisores** | Sofia (Eng. de Segurança), Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos) |
| **Decisão de origem** | D3 (fact-base §1) |
| **Requisitos cobertos** | PRD-FR-06, PRD-FR-10 · **Risco mitigado:** PRD-RISK-02 |

> Escopo desta ADR: registra **uma** decisão — o mecanismo de autenticidade/integridade da entrega outbound. O contrato completo de payload, a lista completa de headers e a matriz de erros `WEBHOOK_*` pertencem ao FDD e não são reproduzidos aqui.

---

## Contexto

O Sistema de Webhooks expõe dados de pedidos para endpoints **fora da nossa infraestrutura**. Como a chamada sai da nossa plataforma e chega a um servidor de terceiro, o cliente precisa conseguir provar, por conta própria, duas coisas: que a requisição **veio realmente de nós** (autenticidade) e que **ninguém adulterou o payload** no caminho (integridade) — levantado pela Sofia na reunião [09:19].

Há um agravante concreto e não hipotético: **já houve cliente que vazou uma secret em log de aplicação dele** [09:22] Diego. Isso torna o vazamento de credencial um risco tratado como certo, não como exceção (PRD-RISK-02), e impõe duas exigências ao desenho: **conter o raio de dano** de um vazamento e permitir **trocar a credencial sem interromper a entrega**.

Vale delimitar: a exigência de **TLS/HTTPS** na URL do webhook é um requisito de transporte tratado à parte (validação Zod, PRD-NFR-03) — a própria Sofia observou que "nem é decisão arquitetural" [09:23]. Esta ADR trata da camada de **aplicação**: a assinatura da mensagem, complementar ao TLS.

Do lado do nosso servidor, a disciplina de não expor segredos em log já existe: o logger Pino aplica `redact` sobre `authorization`, `cookie`, `*.password`, `*.token`, entre outros (`src/shared/logger/index.ts`). O incidente que motiva a rotação ocorreu **do lado do cliente** [09:22], fora do nosso controle — daí a necessidade de um mecanismo de rotação, e não apenas de higiene de logs interna.

---

## Decisão

Assinar cada entrega com **HMAC-SHA256 sobre o corpo do request**, com **secret única por endpoint** e suporte a **rotação com grace period de 24h**. Decisão fechada por Sofia [09:22] e ratificada por Larissa no fechamento [09:48].

| ID | Parâmetro da decisão | Definição | Origem |
|----|----------------------|-----------|--------|
| D3.1 | Algoritmo | HMAC-SHA256 (SHA-256) sobre o corpo do request; padrão de mercado, com biblioteca disponível em todo cliente sério | [09:20][09:22] Sofia |
| D3.2 | Transporte da assinatura | Enviada no header `X-Signature`; o cliente recalcula e compara do lado dele | [09:20] Sofia; [09:44] Diego |
| D3.3 | Escopo da secret | **Única por endpoint**, não uma secret global da plataforma | [09:21] Sofia |
| D3.4 | Geração e entrega | Secret **gerada por nós** e devolvida ao cliente **na criação** do webhook | [09:31] Marcos |
| D3.5 | Rotação | Endpoint de API para o cliente pedir nova secret; a **antiga permanece válida por 24h em paralelo** (grace period) e depois é invalidada | [09:21] Sofia; PRD-FR-06 |
| D3.6 | Identificação do endpoint | Entrega carrega também `X-Webhook-Id`, para o cliente com vários cadastros saber qual endpoint (e qual secret) corresponde à mensagem | [09:44] Sofia |

O detalhamento da mecânica de assinatura/validação durante a janela de grace, do modelo de armazenamento da secret (secret ativa + anterior + expiração da anterior) e do contrato do endpoint de rotação (`POST /webhooks/:id/secret/rotate`) é responsabilidade do **FDD** — aqui fica apenas a decisão e o porquê.

Erros novos ligados a URL e rotação (por exemplo `WEBHOOK_URL_NOT_HTTPS`, `WEBHOOK_SECRET_ROTATION_PENDING`) seguem a família `AppError` já existente (`src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`) com prefixo `WEBHOOK_`, conforme ADR-006 — sem stack nova. A matriz completa fica no FDD.

---

## Alternativas Consideradas

| ID | Alternativa | Por que foi descartada | Origem |
|----|-------------|------------------------|--------|
| RFC-ALT-06 | **Secret global da plataforma** (uma única secret para assinar todas as entregas de todos os clientes) | É operacionalmente mais simples (uma credencial só), mas o raio de dano é **total**: "se vaza uma, vaza tudo". Um único vazamento — exatamente o cenário já observado [09:22] — comprometeria a assinatura de **todos** os clientes de uma vez, exigindo rotação simultânea e coordenada em toda a base. Incompatível com a exigência de conter o raio de dano. | [09:21] Sofia |

A escolha do algoritmo não abriu alternativas concorrentes na reunião: SHA-256 foi adotado diretamente como padrão de mercado (Stripe/GitHub assinam com HMAC-SHA256), com o benefício explícito de que "todo cliente sério tem biblioteca pra isso" [09:20] Sofia.

---

## Consequências

### Positivas

- **Verificação independente pelo cliente.** Com a secret compartilhada e o `X-Signature`, o cliente valida autenticidade e integridade sem sessão, callback ou estado adicional do nosso lado [09:19][09:20].
- **Raio de dano contido.** Secret por endpoint isola o impacto de um vazamento a um único cadastro, em vez de a toda a plataforma [09:21] — mitigação direta de PRD-RISK-02.
- **Rotação sem downtime.** O grace period de 24h com a secret antiga válida em paralelo permite ao cliente migrar seus sistemas sem perder entregas nem sofrer falhas de validação durante a troca [09:21] — resposta concreta ao incidente real de secret vazada em log [09:22].
- **Baixo atrito de integração.** HMAC-SHA256 é padrão de mercado; clientes já têm biblioteca pronta, o que reduz o custo de adoção [09:20].
- **Reuso da base existente.** Os novos erros de URL/rotação estendem a família `AppError` já existente — assim como `InvalidStatusTransitionError` (via `ConflictError`, 409) e `InsufficientStockError` (via `UnprocessableEntityError`, 422) —, capturados pelo error middleware central sem alteração (ADR-006).

### Negativas / Trade-offs

- **Responsabilidade de verificação transferida ao cliente.** O cliente precisa implementar e manter a checagem HMAC corretamente, e — sobretudo — **guardar a secret com segurança**. O próprio risco que motivou a decisão (vazamento em log do cliente [09:22]) continua existindo; a rotação e o isolamento por endpoint **reduzem e circunscrevem** o impacto, mas não o eliminam. A Sofia observou explicitamente que o modelo "joga responsabilidade pro cliente" [09:25].
- **Estado adicional e complexidade na rotação.** Suportar a janela de 24h exige persistir mais de uma secret por endpoint (ativa + anterior com expiração) e tratar a validade em paralelo — complexidade de modelo de dados e de fluxo de assinatura que não existiria com uma secret imutável (detalhe delegado ao FDD, fact-base §12).
- **Gate de segurança no cronograma.** A implementação de HMAC e a geração de secret precisam de revisão dedicada: a Sofia reservou **≥2 dias úteis** para revisar esse código de segurança **antes do deploy** [09:46]. É uma dependência de entrega, embutida nas 3 sprints estimadas [09:46][09:47].

**Trade-off explícito:** aceitamos transferir ao cliente a responsabilidade de verificar a assinatura e de proteger a secret, além de introduzir o estado e a mecânica de rotação, **em troca de** um mecanismo de autenticidade padrão de mercado, de baixo atrito, com raio de dano contido por endpoint e troca de credencial sem interrupção — em vez da secret global (RFC-ALT-06), operacionalmente mais simples porém com raio de dano total a cada vazamento.

---

## Referências

**Reunião técnica (TRANSCRICAO.md):**
- [09:19] Sofia — dados de pedido expostos a endpoint externo; cliente precisa validar origem e integridade.
- [09:20] Sofia — padrão HMAC, assinatura em `X-Signature`, algoritmo SHA-256 como padrão de mercado (respondendo à pergunta de Bruno sobre qual algoritmo, [09:20]).
- [09:21] Sofia — secret única por endpoint (não global); rotação com secret antiga válida por 24h em paralelo.
- [09:22] Diego — motivação real: cliente já vazou secret em log de aplicação.
- [09:22] Sofia — decisão fechada: HMAC-SHA256 sobre o corpo, secret por endpoint, grace period de 24h.
- [09:25] Sofia — o modelo transfere responsabilidade ao cliente.
- [09:31] Marcos — secret gerada por nós e devolvida na criação.
- [09:44] Diego / [09:44] Sofia — headers `X-Signature` e `X-Webhook-Id`.
- [09:46] Sofia — reserva de ≥2 dias úteis para revisão de segurança de HMAC e geração de secret antes do deploy.
- [09:48] Larissa — ratificação da decisão no resumo de fechamento.

**Fact-base canônica:** D3 (§1) · RFC-ALT-06 (§9) · PRD-RISK-02 (§10) · PRD-FR-06, PRD-FR-10 (§4) · modelo de secret (§12).

**Código existente (contexto e reuso):**
- `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` — família `AppError` e suas subclasses (`ConflictError` 409, `UnprocessableEntityError` 422) que os erros `WEBHOOK_*` de URL/rotação estendem (ADR-006).
- `src/shared/logger/index.ts` — Pino com `redact` de campos sensíveis, disciplina interna já existente contra exposição de segredos em log.
- `src/middlewares/validate.middleware.ts` — validação Zod usada para a exigência complementar de HTTPS (PRD-NFR-03), fora do escopo desta ADR.

**Relacionadas:** ADR-002 (worker que executa a assinatura no envio) · ADR-006 (reuso de `AppError`, Pino e error middleware). Contrato de headers/payload e matriz `WEBHOOK_*` completos: **FDD**.
