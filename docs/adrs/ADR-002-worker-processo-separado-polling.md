# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 15 de julho de 2026
- **Autor:** Gabriele Rocha
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior – Plataforma), Marcos (PM), Bruno (Eng. Pleno – Pedidos)
- **Decisão de origem:** D5 (fact-base §1)
- **Escopo:** consumo da outbox definida no [ADR-001](./ADR-001-outbox-no-mysql.md). Este ADR registra **apenas** como a outbox é lida e por quê; o padrão outbox em si é decidido no ADR-001.

---

## Contexto

O padrão Outbox (ADR-001) grava, na mesma transação que muda o status do pedido, uma linha em `webhook_outbox`. Falta decidir **quem lê essa outbox e dispara o HTTP** para o cliente, e com qual reatividade.

Restrições que balizam a decisão:

- Os clientes B2B consideram "tempo real" qualquer entrega **abaixo de 10 segundos** [09:02] Marcos. A leitura da outbox precisa caber folgadamente nessa janela.
- O disparo tem de ficar **fora** da transação de mudança de status — motivo já decidido no ADR-001 (transação já pesada; cliente lento/offline não pode travar nem forçar rollback) [09:04] Bruno.
- O time é pequeno; a solução não deve introduzir infraestrutura nova [09:07] Diego (vide ADR-001, que descarta fila externa).
- A plataforma roda em **Node.js + Express + Prisma + MySQL**. O MySQL **não** oferece `NOTIFY`/`LISTEN` como o Postgres [09:09] Diego.
- Já existe um entry-point de bootstrap consolidado (`src/server.ts`) e uma fábrica de client Prisma (`src/config/database.ts`) que servem de molde.

A questão é: reagir a eventos (trigger/notificação nativa) ou consultar a outbox periodicamente (polling)? E onde esse consumidor deve rodar — junto da API ou isolado?

---

## Decisão

Um **worker dedicado, rodando como processo Node separado**, consome a outbox por **polling em loop a cada 2 segundos**: a cada ciclo busca os eventos pendentes mais antigos, processa e marca o resultado [09:09] Diego.

Parâmetros fechados na reunião:

| Parâmetro | Valor decidido | Origem |
|-----------|----------------|--------|
| Modelo de leitura | Polling em loop | [09:09] Diego |
| Intervalo de polling | **2 segundos** | [09:09] Diego; fechado [09:10] Larissa/Marcos |
| Topologia | **Processo separado** da API | [09:11] Diego |
| Entry-point | Novo `src/worker.ts` + script `npm run worker`, análogo ao `src/server.ts` | [09:11] Larissa |
| Conexão de banco | Instância própria de `PrismaClient` no processo do worker, mesma `DATABASE_URL` | [09:30] Bruno |

**Por que processo separado.** Se o worker vivesse dentro da instância da API, um restart da API derrubaria o worker junto; isolando-o em outro processo, o consumo da outbox sobrevive a reinícios da API [09:11] Diego (requisito PRD-NFR-07).

**Por que 2 segundos.** Um ciclo de 2s garante que, no pior caso, um evento espera no máximo ~2s até ser detectado — bem dentro do teto de 10s exigido [09:02] Marcos. A latência mínima de 2s no pior caso foi explicitamente **aceita** [09:10] Larissa/Marcos.

### Âncoras no código existente

- **`src/server.ts`** — molde do bootstrap. A função `bootstrap()` monta a aplicação via `buildApp({ prisma })`, registra `shutdown` em `SIGINT`/`SIGTERM` e chama `prisma.$disconnect()` no encerramento. O `src/worker.ts` espelha esse formato: inicializa recursos, entra no loop de polling e faz shutdown gracioso do mesmo modo (referência de implementação — o desenho do loop e do processamento pertence ao FDD, não a este ADR).
- **`src/config/database.ts`** — expõe `createPrismaClient()`, fábrica que retorna um `PrismaClient` novo (ajustando `log` por `NODE_ENV`). Como o worker é **outro processo Node**, ele instancia seu próprio client — via essa mesma fábrica — apontando para a **mesma `DATABASE_URL`**, com pool de conexões próprio. `PrismaClient` é por processo, então não há como (nem por que) compartilhar a instância da API [09:30] Bruno.

> Detalhamento de batch, marcação de status, timeout de 10s do HTTP call e formato de payload são **altitude de FDD** e não são repetidos aqui.

---

## Alternativas Consideradas

| ID | Alternativa | Por que foi descartada | Origem |
|----|-------------|------------------------|--------|
| RFC-ALT-03 | **Trigger de banco** para reatividade (em vez de polling) | O MySQL não tem `NOTIFY`/`LISTEN` nativo como o Postgres; uma trigger só executa SQL, **não notifica um processo externo**. Para avisar o worker seria preciso improvisar (escrever em arquivo, bater num endpoint), o que "fica esquisito". Polling de 2s já atende o "abaixo de 10s" com folga. | [09:09] Diego |
| ADR-002-ALT-A | **Worker dentro da própria instância da API** (mesmo processo) | Se a API reinicia, o worker cai junto e o consumo da outbox para. Isolar em processo separado torna a entrega resiliente a restart da API. | [09:11] Diego |

Observação: o disparo **síncrono** dentro do `order.service` também foi descartado (RFC-ALT-01), mas essa decisão pertence ao ADR-001 (padrão Outbox) e não se repete aqui.

---

## Consequências

### Positivas

- **Atende ao requisito de tempo real** com margem: latência de detecção ≤ ~2s contra teto de 10s [09:02][09:10] (PRD-NFR-01).
- **Resiliência operacional:** o worker sobrevive a reinícios da API por ser processo independente [09:11] Diego (PRD-NFR-07).
- **Sem infraestrutura nova:** reaproveita MySQL, Prisma e a stack Node existentes; nenhuma dependência de `NOTIFY`/`LISTEN` ou de broker externo [09:07][09:09].
- **Baixo custo de construção:** `src/worker.ts` reusa o molde de `src/server.ts` e a fábrica `createPrismaClient()` de `src/config/database.ts`; o entry-point novo se encaixa no padrão do projeto (coerente com o ADR-006 de reuso máximo).
- **Simplicidade e previsibilidade:** polling é um mecanismo simples e determinístico, mais fácil de operar e depurar que reatividade improvisada sobre MySQL.

### Negativas (trade-offs assumidos)

- **Latência mínima de 2s no pior caso.** Um evento pode aguardar até o próximo ciclo de polling antes de ser detectado. Trade-off **explicitamente aceito** [09:10] Larissa/Marcos, por caber no teto de 10s.
- **Consulta recorrente à outbox.** Por ser polling (e não reativo), o worker consulta a tabela a cada 2s mesmo sem eventos pendentes. Mitigado por design pela leitura de apenas pendentes em batch pequeno, com índice em status/`created_at` (ver ADR-001); o custo dessa consulta contínua foi o preço aceito para evitar a reatividade improvisada descartada em RFC-ALT-03.
- **Mais um processo para operar e implantar.** Rodar `npm run worker` como processo à parte adiciona uma unidade de deploy/monitoramento além da API — contrapartida direta e desejada do ganho de resiliência.
- **Ordenação depende de worker único.** O desenho assume um único worker consumindo em ordem de `created_at`; escalar para múltiplos workers em paralelo (particionamento por `order_id`/lock) permanece questão em aberto (RFC-OPEN-02) e está fora do escopo desta feature.

---

## Referências

- Decisão D5 e restrições correlatas — fact-base §1 (D5), §5 (PRD-NFR-01, PRD-NFR-07), §9 (RFC-ALT-03).
- Transcrição da reunião:
  - [09:02] Marcos — "tempo real" = abaixo de 10 segundos.
  - [09:09] Diego — polling a cada 2s; trigger de banco descartada (MySQL sem `NOTIFY`/`LISTEN`).
  - [09:10] Larissa / Marcos — decisão fechada: polling 2s, latência mínima de 2s aceita.
  - [09:11] Diego — worker como processo separado; se a API reinicia, não perde o worker.
  - [09:11] Larissa — novo entry-point `src/worker.ts` + `npm run worker`, análogo a `src/server.ts`.
  - [09:11] Diego — mesmo banco e mesma stack, mas **processo separado** (Bruno inicialmente cogitou compartilhar o Prisma client); [09:30] Bruno — instância **nova** de `PrismaClient` (é por processo), mesma `DATABASE_URL`.
- Código real:
  - `src/server.ts` — molde do bootstrap (`bootstrap`/`buildApp`, shutdown gracioso, `prisma.$disconnect()`).
  - `src/config/database.ts` — `createPrismaClient()`, instância própria de `PrismaClient` por processo.
- ADRs relacionados: [ADR-001 — Padrão Outbox no MySQL](./ADR-001-outbox-no-mysql.md) (origem da outbox consumida por este worker); ADR-006 — Reuso dos padrões existentes.
