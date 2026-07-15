# Da Reunião ao Documento — Pacote de Design Docs (Webhooks de Notificação de Pedidos)

Pacote de documentação técnica da feature **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código de um Order Management System (OMS) em produção (Node.js + TypeScript + Express + Prisma + MySQL).

> Este README documenta o **processo de produção**. O enunciado original do desafio está preservado no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia) (e no histórico do git deste fork).

---

## Sobre o desafio

O cenário: uma empresa que opera um OMS em produção decidiu, numa reunião entre tech lead, PM, engenheiros e segurança, construir um sistema de webhooks para notificar clientes B2B quando o status de um pedido muda. A decisão técnica foi tomada, mas **nada foi registrado além da transcrição da call**. A tarefa é transformar essa transcrição (e o código existente) em um pacote acionável de design docs — PRD, RFC, FDD, ADRs, Tracker — em nível suficiente para o time começar a implementar.

O ponto central do desafio não é "gerar documentos com IA", e sim exercer o papel de **maestro**: decidir o que precisa ser produzido, escrever bons prompts, revisar criticamente cada entrega da IA e iterar até o resultado ficar consistente e **100% rastreável** — sem inventar requisitos, decisões ou restrições que não tenham origem na transcrição ou no código. Identificar o que **não** entra (itens descartados ou adiados na reunião) é tão importante quanto identificar o que entra.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel neste desafio |
|-----------|---------------------|
| **Claude Code (Claude Opus 4.8, contexto 1M)** | Ferramenta principal. Leitura do código e da transcrição, construção da fact-base canônica, orquestração dos agentes de redação/verificação e aplicação das correções finais. |
| **Orquestração multiagente (workflow do Claude Code)** | Um pipeline `draft → verify` com um agente por documento (11 no total). Cada redator produziu um documento a partir da mesma fonte da verdade; cada verificador adversarial checou o documento contra a transcrição e o código. |
| **Subagentes de verificação adversarial** | Agentes com a instrução explícita de "assumir que o documento contém erros e encontrá-los" — caçaram alucinações, citações imprecisas, referências de código inexistentes e violações de altitude entre documentos. |

Toda a produção foi feita com Claude Code no terminal, com o código e a transcrição disponíveis como contexto direto (sem colar trechos manualmente).

---

## Workflow adotado

A ordem seguiu a sugestão do desafio (decisões primeiro, documento de produto por último), com uma etapa inicial de blindagem contra alucinação:

1. **Contextualização.** Li integralmente `TRANSCRICAO.md` e mapeei o código real que a feature toca: `src/modules/orders/order.service.ts` (`changeStatus`), a máquina de estados (`order.status.ts`), a família de erros (`src/shared/errors/*`), o error middleware, o `authenticate`/`requireRole`, o logger Pino, os entry-points (`server.ts`/`database.ts`) e o `schema.prisma`.

2. **Fact-base canônica (a espinha anti-alucinação).** Antes de escrever qualquer documento, consolidei numa **fonte da verdade** única todas as decisões, requisitos funcionais e não funcionais, itens fora de escopo, questões em aberto, riscos, contratos e pontos de integração — **cada um com sua citação** (`[hh:mm] Falante` da transcrição ou caminho de arquivo do código). A regra imposta a todos os agentes: *se não está na fact-base, não existe*.

3. **ADRs, RFC e FDD em paralelo (draft).** Um agente redator por documento (PRD, RFC, FDD e os 8 ADRs), todos consumindo **a mesma fact-base** — o que garante consistência de IDs, endpoints, códigos de erro e citações entre documentos. Cada agente recebeu a **altitude** do seu documento e uma lista explícita do que **não** incluir (ex.: o RFC não repete o detalhe do FDD).

4. **Verificação adversarial (verify).** Cada rascunho passou por um verificador cético que releu o documento contra a transcrição e o código e devolveu achados estruturados: alucinações, timestamps errados, arquivos inexistentes, violações de altitude e seções obrigatórias ausentes.

5. **Correção dirigida.** Revisei os achados (3 documentos passaram limpos; 8 tiveram ajustes menores) e apliquei as correções manualmente, uma a uma, conferindo cada uma contra a fonte.

6. **PRD, Tracker e este README.** O PRD consolidou o material já formalizado. O **Tracker** foi montado ao final, cruzando cada ID dos documentos com sua citação. O README ficou por último, com o processo já completo.

Diagrama do processo:

```
Transcrição + Código
        │
        ▼
  Fact-base canônica  ──(fonte da verdade única, com citações)
        │
        ├──────────────► 11 agentes redatores (PRD, RFC, FDD, ADR-001..008)   [draft]
        │                        │
        │                        ▼
        │                11 verificadores adversariais                          [verify]
        │                        │
        ▼                        ▼
  Correção dirigida ◄──── achados estruturados (3 PASS, 8 MINOR_FIX)
        │
        ▼
  Tracker  ─►  README
```

---

## Prompts customizados

### 1. Regras invioláveis impostas a todo agente redator (a "constituição" anti-alucinação)

Preâmbulo comum a todos os redatores, garantindo rastreabilidade e respeito à altitude de cada documento:

```text
Você é engenheiro de software SÊNIOR atuando como redator técnico. Produz UM documento
de um pacote de design docs para a feature "Sistema de Webhooks de Notificação de Pedidos".

MATERIAL-FONTE — leia ANTES de escrever, nesta ordem:
1. Fact-base canônica (FONTE DA VERDADE, com citações rastreáveis)
2. Transcrição literal da reunião (TRANSCRICAO.md)
3. Código real (leia os arquivos que precisar citar)

REGRAS INVIOLÁVEIS:
- Rastreabilidade total. Todo requisito/decisão/restrição/risco vem da fact-base
  (que cita [hh:mm] Falante ou caminho de arquivo). NÃO INVENTE nada.
- Onde a fact-base traz [hh:mm] Falante, cite inline. Nunca invente timestamps.
- Nunca cite um arquivo de código inexistente. Use apenas os caminhos reais.
- Itens explicitamente descartados/adiados na reunião JAMAIS podem aparecer como requisito.
- Respeite a ALTITUDE deste documento. Duplicar o detalhe de outro documento é ERRO.
```

### 2. Verificador adversarial (o "advogado do diabo" da documentação)

Rodado sobre cada documento depois de escrito, para caçar o que a IA inventou ou atribuiu errado:

```text
Você é um REVISOR ADVERSARIAL de documentação técnica. Assuma que o documento contém
erros e sua missão é encontrá-los — seja cético e específico.

Verifique, para este documento:
1. ALUCINAÇÕES: toda afirmação factual (requisito, número, prazo, decisão, nome,
   timestamp) tem lastro na fact-base/transcrição/código? Itens descartados/adiados
   (email de aviso, rate limiting, dashboard, exactly-once) NÃO podem aparecer como
   requisito vigente.
2. TIMESTAMPS: todo [hh:mm] citado existe mesmo e corresponde ao falante certo?
3. REFERÊNCIAS DE CÓDIGO: todo caminho de arquivo citado existe no repositório?
4. ALTITUDE: há conteúdo no nível de detalhe errado? (PRD sem payload/DDL/matriz;
   RFC conciso; FDD com o detalhe; ADR = uma decisão só.)
5. SEÇÕES OBRIGATÓRIAS e CRITÉRIOS DE ACEITE do desafio para este tipo de doc.

Retorne achados estruturados; cada achado deve citar o trecho problemático.
```

Além desses, o prompt de construção da **fact-base** pediu explicitamente para *separar o que foi decidido, do que foi descartado, do que foi adiado* — porque essa filtragem é o que impede itens como "e-mail de aviso" ou "dashboard" de vazarem para os documentos como se fossem requisitos.

---

## Iterações e ajustes

Foram cerca de **5 ciclos principais**: (1) extração da fact-base, (2) redação paralela, (3) verificação adversarial, (4) correção dirigida e (5) consolidação/tracker. A verificação adversarial foi o que mais elevou a qualidade — ela pegou erros sutis que passariam despercebidos numa leitura casual. Os ajustes concretos mais relevantes:

- **ADR-004 — hierarquia de erros errada (erro factual contra o código).** O rascunho afirmava que `InsufficientStockError` seguia a família `ConflictError`. Conferindo `src/shared/errors/http-errors.ts`, ela estende `UnprocessableEntityError` (422); só `InvalidStatusTransitionError` estende `ConflictError` (409). Corrigido para refletir o código real. *Lição: mesmo detalhe "de apoio" precisa bater com o código.*

- **ADR-002 — atribuição de fala invertida.** O rascunho creditava a `[09:11] Bruno` a ideia de "PrismaClient próprio por processo" — mas nesse minuto Bruno defendeu o **oposto** ("usar o mesmo Prisma client"); a instância separada só é decidida em `[09:30]`. Reatribuí a citação ao timestamp correto.

- **ADR-003 — condição de falha inventada.** O rascunho dizia que uma entrega falha por "resposta não-2xx", o que **não aparece na transcrição** (a reunião só definiu o timeout de 10s como falha). Reescrevi como convenção de implementação (detalhada no FDD), sem atribuí-la a uma fala que não existe.

- **RFC — vazamento de altitude para o FDD.** O diagrama e a seção de impacto expunham nomes de tabela, estados de enum (`status=PENDING`) e a assinatura de `publishWebhookEvent(tx, ...)` — detalhe que o próprio RFC declara pertencer ao FDD. Generalizei o diagrama e removi a assinatura.

- **PRD — citação esticada.** A estratégia de testes citava `[09:29] Bruno` como origem do "Vitest", mas naquele momento Bruno falava do **logger**, não de testes. Reancorei em `package.json`/`tests/` (código) + a decisão de reuso `[09:30] Larissa`.

- **ADR-008 — inferência sem lastro.** Um trade-off usava `[09:14] Marcos` (que era sobre *ordenação global*) para justificar "não fazer backfill histórico". Removi a citação indevida e reescrevi o ponto como consequência assumida do desenho.

O padrão dos erros é revelador: a IA raramente errou nas **decisões**; ela errou nas **atribuições** (esticar um timestamp para cobrir uma afirmação vizinha) e em **detalhes técnicos plausíveis mas não-ditos**. É exatamente contra esse tipo de erro que a fact-base e o tracker existem.

---

## Como navegar a entrega

Ordem de leitura sugerida (do "porquê" ao "como"):

1. **[`docs/PRD.md`](docs/PRD.md)** — o problema, o público, o escopo (incluso e **fora**) e as métricas de sucesso. *Por quê e o quê.*
2. **[`docs/RFC.md`](docs/RFC.md)** — a proposta técnica em alto nível, as alternativas descartadas e as questões em aberto. *Como pretendemos resolver.*
3. **[`docs/adrs/`](docs/adrs/)** — as 8 decisões arquiteturais, uma por arquivo (Outbox, worker/polling, retry+DLQ, HMAC, at-least-once, reuso, snapshot, filtragem). *Por que decidimos exatamente assim.*
4. **[`docs/FDD.md`](docs/FDD.md)** — o desenho de implementação: fluxos, contratos de API, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a **integração com o código existente**. *Como construir, em detalhe.*
5. **[`docs/TRACKER.md`](docs/TRACKER.md)** — a rastreabilidade de cada item à transcrição ou ao código. *De onde veio cada coisa.*

Estrutura dos arquivos entregues:

```
.
├── README.md                  ← este documento (processo)
├── TRANSCRICAO.md             ← fonte primária (não alterada)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-processo-separado-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-x-event-id.md
│       ├── ADR-006-reuso-padroes-existentes.md
│       ├── ADR-007-snapshot-payload-na-outbox.md
│       └── ADR-008-filtragem-eventos-na-insercao.md
└── src/ · prisma/ · tests/    ← código do boilerplate (contexto; não alterado)
```

> A entrega é **puramente documental**: nenhum arquivo em `src/`, `prisma/`, `tests/` ou de configuração foi alterado. O código serviu apenas de contexto e referência.
