# Architectural Decision Records (ADRs)

Este diretório armazena os Architectural Decision Records (ADRs) da feature de Sistema de Webhooks de Notificação de Pedidos.

## Índice de Decisões

| ID | Título | Status | Data |
| --- | --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Adoção do Padrão Outbox no MySQL para Enfileiramento de Webhooks | Aceito | 2026-07-22 |
| [ADR-002](./ADR-002-worker-separado-em-polling.md) | Worker em Processo Separado em Polling a Cada 2 Segundos | Aceito | 2026-07-22 |
| [ADR-003](./ADR-003-retry-e-dead-letter-queue.md) | Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ) | Aceito | 2026-07-22 |
| [ADR-004](./ADR-004-autenticacao-hmac-e-rotacao-de-secrets.md) | Autenticação via HMAC-SHA256, Secret por Endpoint e Rotação com Grace Period | Aceito | 2026-07-22 |
| [ADR-005](./ADR-005-entrega-at-least-once-e-idempotencia.md) | Garantia de Entrega At-Least-Once e Idempotência via Header X-Event-Id | Aceito | 2026-07-22 |
