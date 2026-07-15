# ADR-005 — Entrega at-least-once com idempotência via X-Event-Id

- **Status:** Aceito
- **Data:** 15 de julho de 2026
- **Autor:** Gabriele Rocha
- **Decisores:** Diego (Eng. Sênior – Plataforma, propositor), Larissa (Tech Lead, fechou a decisão), Marcos (PM), Sofia (Eng. de Segurança), Bruno (Eng. Pleno – Pedidos)
- **Decisão de origem:** D4 (fact-base §1)
- **ADRs relacionados:** ADR-001 (Outbox no MySQL), ADR-002 (worker em polling), ADR-003 (retry com backoff e DLQ)

---

## Contexto

O sistema de webhooks entrega eventos de mudança de status de pedido de forma assíncrona: a mudança grava um evento na outbox dentro da transação de status (ADR-001) e um worker separado, em polling, lê os pendentes e dispara a chamada HTTP ao endpoint do cliente (ADR-002). Em caso de falha, o evento é reenviado com backoff exponencial por até 5 tentativas antes de ir para a DLQ (ADR-003).

Esse desenho torna a **duplicação de entrega inevitável na prática**. O worker considera falha qualquer resposta que não chegue dentro do timeout de 10s [09:42]. Se o cliente processa o evento mas a resposta se perde ou demora além do timeout, o worker marca a tentativa como falha e reenvia o mesmo evento — o cliente recebe duas vezes. Precisamos definir qual garantia de entrega oferecer e como o cliente distingue um reenvio de um evento novo.

Duas garantias estavam sobre a mesa: **exactly-once** (o cliente nunca recebe duplicata) e **at-least-once** (o cliente pode receber a mesma notificação mais de uma vez e precisa lidar com isso). A escolha foi discutida a partir de [09:24].

Fatores de contexto relevantes para a decisão:

- Os clientes nunca pediram garantia de ordenação global; só querem saber se cada pedido mudou [09:14] Marcos — o que reduz a exigência sobre a semântica de entrega.
- O padrão de mercado para webhooks (Stripe, GitHub) é at-least-once [09:25] Diego.
- Marcos se comprometeu a documentar a semântica de forma destacada no portal do desenvolvedor [09:26][09:40], viabilizando que o cliente saiba que precisa deduplicar.

## Decisão

Adotar **entrega at-least-once**: o worker garante que cada evento é entregue **ao menos uma vez**, aceitando que, em cenários de retry, o cliente possa recebê-lo mais de uma vez [09:24] Diego. Não haverá tentativa de suprimir duplicatas do nosso lado.

A **idempotência é delegada ao cliente** por meio do header **`X-Event-Id`**, que carrega o UUID do evento. Esse UUID é o próprio `id` da linha da outbox, gerado no momento da inserção do evento (dentro da transação de mudança de status) e seguindo a convenção de identificadores `uuid @db.Char(36)` já usada em todo o projeto [09:51] Larissa (ver `prisma/schema.prisma`). O identificador é **único por evento e estável entre retransmissões**: todas as tentativas de reenvio do mesmo evento carregam o mesmo `X-Event-Id`, permitindo que o cliente deduplique de forma determinística do lado dele [09:25] Diego.

O `X-Event-Id` acompanha os demais headers de rastreio no request (assinatura, timestamp, id do endpoint), definidos em [09:44] — cujo contrato completo é responsabilidade do FDD, não deste ADR. A decisão at-least-once com `X-Event-Id` foi fechada em [09:26] Larissa.

## Alternativas Consideradas

### Exactly-once delivery (RFC-ALT-05) — descartada

Garantir que o cliente nunca receba um evento duplicado. Exigiria coordenação entre os dois lados, tornando a implementação **muito mais complexa** [09:25] Diego. Foi descartada porque:

- A complexidade de coordenação bidirecional não se justifica para um time pequeno, na mesma linha do que motivou evitar infraestrutura extra em ADR-001/ADR-002.
- At-least-once com um id de evento resolve 99% dos casos e é o comportamento consolidado de mercado (Stripe, GitHub) [09:25] Diego.
- A responsabilidade de deduplicar cabe ao cliente com custo baixo (comparar um UUID), e será documentada no portal do desenvolvedor [09:26] Marcos.

## Consequências

### Positivas

- **Simplicidade e alinhamento com o mercado.** Sem protocolo de confirmação bidirecional; reproduz o padrão que integradores B2B já conhecem de Stripe e GitHub [09:25], reduzindo atrito de integração.
- **Nenhuma perda de evento.** A garantia combina de forma natural com o retry/backoff (ADR-003): preferimos reenviar (e arriscar duplicata) a arriscar perder uma notificação.
- **Dedup determinística no cliente.** Como o `X-Event-Id` é único por evento e idêntico em todas as retransmissões, a deduplicação do lado do cliente é trivial (chave única / "já vi este id?").
- **Custo de implementação baixo.** O identificador reaproveita o `id` (uuid) da própria linha da outbox [09:51], sem gerar nem persistir nada novo só para o header.

### Negativas / Trade-off explícito

- **A responsabilidade de idempotência recai sobre o cliente** [09:25] Sofia. Um cliente que não implemente a deduplicação por `X-Event-Id` processará eventos repetidos e poderá gerar efeitos colaterais duplicados. **Mitigação:** Marcos documentará a semântica at-least-once de forma destacada no portal do desenvolvedor [09:26][09:40].
- **Não oferecemos exactly-once.** Integrações que precisem de garantia forte de unicidade terão de construí-la sobre o `X-Event-Id`; não entregamos essa garantia no protocolo.

**Trade-off aceito:** trocamos a garantia de unicidade (exactly-once) — e o custo de coordenação que ela exigiria de ambos os lados — por um protocolo simples que nunca perde eventos, empurrando ao cliente o custo baixo e localizado de deduplicar por um id estável.

## Referências

- Transcrição da reunião (`TRANSCRICAO.md`):
  - [09:14] Marcos — clientes só querem saber se cada pedido mudou; não pedem ordenação global.
  - [09:24] Diego — garantia at-least-once; cliente pode receber o mesmo evento duas vezes.
  - [09:25] Diego — `X-Event-Id` (UUID do evento) para dedup do lado do cliente; exactly-once descartada por coordenação complexa; referência Stripe/GitHub.
  - [09:25] Sofia — a decisão transfere responsabilidade ao cliente.
  - [09:26] Marcos — documentará a semântica no portal do desenvolvedor.
  - [09:26] Larissa — decisão fechada: at-least-once com `X-Event-Id`.
  - [09:42] Diego — timeout de 10s do HTTP call (mecanismo que pode gerar reenvio/duplicata).
  - [09:44] Diego — `X-Event-Id` entre os headers do request.
  - [09:51] Larissa — id da outbox = UUID (padrão do projeto).
- Código: `prisma/schema.prisma` — convenção de id `uuid @default(uuid()) @db.Char(36)` reusada como `X-Event-Id`.
- Fact-base canônica: §1 (D4), §9 (RFC-ALT-05), §5 (PRD-NFR-06), §3 (headers), §12 (modelo `webhook_outbox`).
- Decisões relacionadas: ADR-001, ADR-002, ADR-003.
