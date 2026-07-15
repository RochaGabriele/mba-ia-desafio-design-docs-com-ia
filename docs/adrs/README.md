# Architectural Decision Records (ADRs)

Este diretório armazena os ADRs da feature **Sistema de Webhooks de Notificação de Pedidos**. Cada arquivo registra **uma** decisão arquitetural no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências), com citações rastreáveis à reunião (`[hh:mm] Falante`) e/ou ao código.

Nomenclatura: `ADR-NNN-titulo-em-kebab-case.md`.

## Índice

| ADR | Decisão | Referencia código real |
|-----|---------|:----------------------:|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL com transação atômica | ✅ |
| [ADR-002](ADR-002-worker-processo-separado-polling.md) | Worker em processo separado com polling de 2 s | ✅ |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e Dead Letter Queue | ✅ |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação | ✅ |
| [ADR-005](ADR-005-at-least-once-x-event-id.md) | Entrega at-least-once com X-Event-Id | ✅ |
| [ADR-006](ADR-006-reuso-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto | ✅ |
| [ADR-007](ADR-007-snapshot-payload-na-outbox.md) | Snapshot do payload renderizado na inserção da outbox | ✅ |
| [ADR-008](ADR-008-filtragem-eventos-na-insercao.md) | Filtragem de eventos por assinatura na inserção da outbox | ✅ |

As decisões 001–006 cobrem as 6 decisões principais da reunião; 007 e 008 registram decisões secundárias fechadas no fechamento da call. A rastreabilidade item a item está em [`../TRACKER.md`](../TRACKER.md).
