# ADR-006 — Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** 15 de julho de 2026
- **Autora:** Gabriele Rocha
- **Decisores:** Bruno (Eng. Pleno – Pedidos, proponente), Larissa (Tech Lead, decisora), Diego (Eng. Sênior – Plataforma). Demais participantes da reunião: Marcos (PM), Sofia (Eng. de Segurança).
- **Decisão relacionada na fact-base:** D6

> **Altitude deste documento:** o ADR registra *a decisão de reusar* e o *porquê*. O contrato de payload, os headers e a matriz completa de erros `WEBHOOK_*` (com status codes) pertencem ao FDD e são apenas referenciados aqui. A mecânica de fiação de cada ponto de integração está em `FDD-INT-01..08`.

---

## Contexto

O Sistema de Webhooks é um módulo novo dentro de um OMS **já em produção** (Node.js + TypeScript + Express + Prisma + MySQL). O projeto tem convenções maduras e consistentes entre domínios, e o time é pequeno, com prazo apertado — estimativa de 3 sprints incluindo a revisão de segurança da Sofia [09:46] e entrega alvo para fim de novembro [09:45] Marcos.

Ao abrir o bloco de estrutura de código, Bruno observou que a codebase já tem um padrão claro: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas [09:27]. Além disso, já existem, e são usados por todo o projeto:

- uma hierarquia de erros com a classe base `AppError` (que carrega `statusCode`, `errorCode` e `details`) e classes específicas como `InvalidStatusTransitionError` e `InsufficientStockError`, com códigos no formato `INVALID_STATUS_TRANSITION` / `INSUFFICIENT_STOCK` — `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`;
- um **error middleware centralizado** que já trata `AppError`, `ZodError` e erros do Prisma — `src/middlewares/error.middleware.ts`;
- o logger **Pino** já configurado no projeto inteiro, com redação de campos sensíveis — `src/shared/logger/index.ts`;
- um middleware de validação por schemas **Zod** — `src/middlewares/validate.middleware.ts`;
- middlewares de autenticação e autorização `authenticate` e `requireRole` — `src/middlewares/auth.middleware.ts`.

A pergunta de design é: o módulo de webhooks deve introduzir abstrações próprias (nova hierarquia de erros, logger alternativo, estrutura de módulo própria) ou espelhar o que já existe? Bruno foi direto sobre o logger — "não vamos botar nada novo" [09:29] — e Larissa fechou o bloco com "reuso máximo do que já existe" [09:30].

---

## Decisão

**Reusar ao máximo os padrões e a infraestrutura compartilhada já existentes no projeto.** O módulo de webhooks é escrito como "mais um módulo" da codebase, sem introduzir stack, framework ou abstração nova. Concretamente, ficam reusados os seguintes ativos, e nada equivalente é recriado:

| ID | Padrão / ativo reusado | Arquivo real (molde) | O que o módulo webhook faz | Origem |
|----|------------------------|----------------------|----------------------------|--------|
| ADR-006-R01 | Classe base de erro `AppError` | `src/shared/errors/app-error.ts` | Erros do webhook estendem `AppError`, herdando `statusCode`/`errorCode`/`details` | [09:28] Bruno |
| ADR-006-R02 | Classes de erro específicas como molde | `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError`, `InsufficientStockError`) | Novas subclasses `WEBHOOK_*` seguem o mesmo padrão de mensagem + `errorCode` + `details` | [09:28] Bruno |
| ADR-006-R03 | Prefixo de código de erro por módulo | (convenção `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | Todo código de erro do módulo usa o prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) | [09:28] Bruno, [09:29] Larissa |
| ADR-006-R04 | Error middleware centralizado | `src/middlewares/error.middleware.ts` | Reusado **sem alteração**: já trata `AppError`/`ZodError`/`Prisma`, então captura os erros `WEBHOOK_*` automaticamente | [09:29] Bruno |
| ADR-006-R05 | Logger Pino | `src/shared/logger/index.ts` | Módulo e worker usam o `logger` existente; nenhuma stack de observabilidade nova | [09:29] Bruno |
| ADR-006-R06 | Validação por schemas Zod | `src/middlewares/validate.middleware.ts` + `src/modules/customers/customer.schemas.ts` | Schemas Zod do webhook e uso do `validate({ body, query, params })` seguindo o padrão dos módulos | [09:27][09:30] Bruno/Larissa |
| ADR-006-R07 | AuthN/AuthZ `authenticate` + `requireRole` | `src/middlewares/auth.middleware.ts` | `authenticate` no CRUD; `requireRole('ADMIN')` no replay de DLQ | [09:36] Sofia/Larissa |
| ADR-006-R08 | Estrutura de módulo por domínio | `src/modules/customers` (controller/service/repository/routes/schemas) | Novo `src/modules/webhooks` espelha a mesma estrutura | [09:27] Bruno |

A codificação exata dos códigos de erro (status HTTP, quando cada um dispara) fica na matriz do FDD; aqui listamos apenas os exemplos que Bruno citou na reunião. A fiação concreta de cada ponto (registro de rotas, chamada dentro da transação de status, instanciação do `PrismaClient` do worker) está detalhada no FDD como `FDD-INT-01..08` e não é reproduzida neste ADR.

> Nota de escopo: a instanciação do `PrismaClient` em processo próprio pelo worker e o entry-point `src/worker.ts` são reuso da mesma stack, mas a decisão de **processo separado** é de ADR-002; a garantia at-least-once com `X-Event-Id` é de ADR-005. Este ADR trata só do reuso de padrões de aplicação.

---

## Alternativas Consideradas

### Alternativa A — Introduzir stack/abstração nova para o módulo (DESCARTADA)

Criar abstrações próprias do webhook: uma hierarquia de erros independente de `AppError`, um logger/observabilidade próprio para o worker e uma estrutura de módulo customizada (por ser um módulo "diferente", parte HTTP e parte worker).

**Por que foi descartada:** o time é pequeno e a stack existente já resolve os mesmos problemas. Bruno rejeitou explicitamente o caminho de novidade no logger — "não vamos botar nada novo" [09:29] — e Larissa fechou pelo "reuso máximo" [09:30]. Introduzir camadas novas seria overengineering, quebraria a consistência entre módulos, aumentaria a superfície de manutenção e de bugs, e comprometeria o prazo de 3 sprints [09:46]. O ganho hipotético (abstrações "sob medida" para webhook) não se justifica frente ao custo.

### Alternativa B — Hierarquia de erros do webhook desacoplada de `AppError`, apenas mapeada no middleware (DESCARTADA)

Manter erros de webhook em classes próprias e adaptar o `error.middleware.ts` para reconhecê-las.

**Por que foi descartada:** exigiria **alterar** o middleware central, que hoje trata os erros de forma polimórfica via `instanceof AppError` — reusá-lo sem tocar é justamente o benefício apontado por Bruno [09:29]. Estender `AppError` (Alternativa escolhida) faz os erros `WEBHOOK_*` serem capturados sem nenhuma mudança no caminho de erro compartilhado.

---

## Consequências

### Positivas

- **Consistência total com a codebase.** Qualquer engenheiro que conhece `src/modules/customers` entende o módulo de webhooks; curva de aprendizado ~zero.
- **Zero código novo no caminho de erro HTTP.** Como as subclasses estendem `AppError`, o `error.middleware.ts` serializa os erros `WEBHOOK_*` automaticamente (`{ error: { code, message, details } }`) sem alteração — reforço direto de [09:29] Bruno.
- **Menos infraestrutura e menos superfície de bug.** Nada de logger, framework de validação ou hierarquia de erro paralelos; observabilidade uniforme via Pino.
- **Revisão facilitada.** A revisão de segurança da Sofia [09:46] e a revisão de design com Bruno/Diego [09:50] incidem sobre padrões já conhecidos, concentrando o esforço no que é realmente novo (HMAC, geração/rotação de secret — ADR-004).
- **Cabe no prazo.** Reuso é o que torna a estimativa de 3 sprints realista [09:46].

### Negativas / Trade-offs

- **Acoplamento à infraestrutura compartilhada.** O módulo fica amarrado às abstrações atuais (`AppError`, error middleware, Pino, `validate`); uma evolução dessas peças impacta o webhook junto. Trade-off aceito em troca de consistência e velocidade.
- **O error middleware cobre só o caminho HTTP, não o worker.** `src/middlewares/error.middleware.ts` roda no pipeline do Express; erros internos do worker (ex.: `WEBHOOK_DELIVERY_TIMEOUT`, timeout de 10s [09:42]) **não** passam por ele — são tratados no processo do worker via Pino e DLQ, não viram resposta HTTP. Reusar o middleware não elimina a necessidade de tratamento de erro próprio no worker.
- **A redação do Pino precisa ser estendida para secrets do webhook.** O `redact` atual cobre `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken` (`src/shared/logger/index.ts`), mas **não** um campo de `secret` de webhook. Como já houve cliente que vazou secret em log [09:22] Diego, reusar o logger "como está" não basta: o caminho que loga a secret precisa entrar na lista de `redactPaths` (detalhe de implementação a tratar no FDD/ADR-004).
- **`customer_id` não vem do JWT reusado.** Reusar `authenticate` traz o JWT do operador, não do cliente; portanto o `customer_id` precisa ser passado no body/path e não pode ser inferido do token [09:32]. Consequência conhecida do reuso do middleware de auth, registrada como decisão pequena no PRD/FDD.

---

## Referências

**Reunião (TRANSCRICAO.md):**
- [09:27] Bruno — padrão de módulos em `src/modules` (controller/service/repository/routes/schemas); webhook segue igual.
- [09:28] Bruno — reuso de `AppError` e das classes específicas (`InsufficientStockError`, `InvalidStatusTransitionError`) como molde; exemplos de códigos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`.
- [09:29] Larissa — prefixo `WEBHOOK_` para todos os códigos do módulo.
- [09:29] Bruno — logger Pino já no projeto ("não vamos botar nada novo"); error middleware centralizado trata AppError/Zod/Prisma sem alteração.
- [09:30] Larissa — decisão fechada: reuso máximo (AppError, Pino, error middleware, padrão de módulos, schemas Zod, códigos de erro). No mesmo minuto, Bruno tratou do `PrismaClient` separado por processo (ver ADR-002).
- [09:36] Sofia/Larissa — reuso de `requireRole('ADMIN')` no replay de DLQ.
- [09:32] Bruno/Marcos/Larissa — `customer_id` não vem do JWT.
- [09:42] Diego — timeout de 10s no worker (contexto do erro de entrega).
- [09:22] Diego — histórico de vazamento de secret em log (contexto da redação).
- [09:46] Sofia/Larissa — estimativa de 3 sprints com revisão de segurança.

**Código (ganchos reais):**
- `src/shared/errors/app-error.ts` — classe base `AppError(message, statusCode, errorCode, details)`.
- `src/shared/errors/http-errors.ts` — `InvalidStatusTransitionError`, `InsufficientStockError` (molde das subclasses `WEBHOOK_*`).
- `src/middlewares/error.middleware.ts` — trata `AppError`/`ZodError`/`Prisma` centralizadamente.
- `src/shared/logger/index.ts` — Pino com `redact`.
- `src/middlewares/validate.middleware.ts` — validação por schemas Zod.
- `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`.
- `src/modules/customers/*` (`customer.routes.ts`, `customer.schemas.ts`, etc.) — molde da estrutura de módulo.

**ADRs e documentos relacionados:** ADR-002 (worker em processo separado), ADR-004 (HMAC-SHA256 e secret por endpoint), ADR-005 (at-least-once com `X-Event-Id`); pontos de integração detalhados no FDD (`FDD-INT-01..08`) e matriz completa de erros `WEBHOOK_*` (`FDD-ERR-*`).
