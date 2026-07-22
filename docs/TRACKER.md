# Matriz de Rastreabilidade (Tracker)

Este documento fornece a rastreabilidade completa entre os itens definidos nos Design Docs (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/`) e as suas respectivas fontes de origem, sejam elas declarações factuais na transcrição da reunião técnica (`TRANSCRICAO.md`) ou arquivos do código-fonte existente da aplicação OMS (`src/` e `prisma/`).

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| `PRD-OBJ-01` | `docs/PRD.md` | Objetivo Quantitativo | Latência de entrega de webhooks inferior a 10 segundos | `TRANSCRICAO` | `[09:02] Marcos` |
| `PRD-FR-01` | `docs/PRD.md` | Requisito Funcional | Cadastro e configuração de endpoints HTTP B2B com secret gerada | `TRANSCRICAO` | `[09:31] Marcos` |
| `PRD-FR-02` | `docs/PRD.md` | Requisito Funcional | Enfileiramento atômico do evento na outbox na mudança de status | `TRANSCRICAO` | `[09:06] Diego` |
| `PRD-FR-03` | `docs/PRD.md` | Requisito Funcional | Worker em processo separado realizando polling a cada 2s | `TRANSCRICAO` | `[09:08] Diego` |
| `PRD-FR-04` | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature com secret por endpoint | `TRANSCRICAO` | `[09:20] Sofia` |
| `PRD-FR-05` | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace period de 24 horas em paralelo | `TRANSCRICAO` | `[09:21] Sofia` |
| `PRD-FR-06` | `docs/PRD.md` | Requisito Funcional | Retry com 5 tentativas e backoff exponencial (1m, 5m, 30m, 2h, 12h) | `TRANSCRICAO` | `[09:17] Diego` |
| `PRD-FR-07` | `docs/PRD.md` | Requisito Funcional | Transferência para DLQ após 5 falhas e replay manual por ADMIN | `TRANSCRICAO` | `[09:18] Diego` |
| `PRD-FR-08` | `docs/PRD.md` | Requisito Funcional | Endpoint GET /webhooks/:id/deliveries para consulta de histórico | `TRANSCRICAO` | `[09:34] Marcos` |
| `PRD-ESCOPO-01` | `docs/PRD.md` | Fora de Escopo | Envio de e-mail de notificação em falhas consecutivas | `TRANSCRICAO` | `[09:37] Larissa` |
| `PRD-ESCOPO-02` | `docs/PRD.md` | Fora de Escopo | Dashboard / Painel visual de interface gráfica no frontend | `TRANSCRICAO` | `[09:40] Larissa` |
| `PRD-ESCOPO-03` | `docs/PRD.md` | Fora de Escopo | Rate limiting de saída por cliente no worker | `TRANSCRICAO` | `[09:39] Larissa` |
| `PRD-RNF-01` | `docs/PRD.md` | Requisito Não Funcional | Protocolo HTTPS obrigatório para a URL do webhook | `TRANSCRICAO` | `[09:23] Sofia` |
| `PRD-RNF-02` | `docs/PRD.md` | Requisito Não Funcional | Limite máximo de payload de 64KB por evento | `TRANSCRICAO` | `[09:24] Diego` |
| `PRD-RNF-03` | `docs/PRD.md` | Requisito Não Funcional | Header X-Event-Id com UUID v4 para deduplicação no cliente | `TRANSCRICAO` | `[09:25] Diego` |
| `PRD-DEC-01` | `docs/PRD.md` | Decisão / Trade-off | Padrão Outbox no MySQL garante consistência atômica sem nova infra | `TRANSCRICAO` | `[09:06] Diego` |
| `PRD-DEC-02` | `docs/PRD.md` | Decisão / Trade-off | Worker em processo separado evita perda em reinicialização da API | `TRANSCRICAO` | `[09:11] Diego` |
| `PRD-DEC-03` | `docs/PRD.md` | Decisão / Trade-off | Rotação de secret com grace period de 24h evita rejeição | `TRANSCRICAO` | `[09:21] Sofia` |
| `PRD-RISCO-01` | `docs/PRD.md` | Risco e Mitigação | Perda do cliente Atlas Comercial por atraso na entrega | `TRANSCRICAO` | `[09:00] Marcos` |
| `PRD-RISCO-02` | `docs/PRD.md` | Risco e Mitigação | Trava do worker por endpoints de clientes lentos ou inacessíveis | `TRANSCRICAO` | `[09:42] Diego` |
| `PRD-CRIT-01` | `docs/PRD.md` | Critério de Aceitação | Latência total de notificação entregue em menos de 10 segundos | `TRANSCRICAO` | `[09:02] Marcos` |
| `PRD-CRIT-02` | `docs/PRD.md` | Critério de Aceitação | Replay manual da DLQ funcional e protegido pela role ADMIN | `TRANSCRICAO` | `[09:36] Sofia` |
| `RFC-ALT-01` | `docs/RFC.md` | Alternativa Descartada | Disparo HTTP síncrono no OrderService travaria a API | `TRANSCRICAO` | `[09:04] Bruno` |
| `RFC-ALT-02` | `docs/RFC.md` | Alternativa Descartada | Mensageria dedicada (Redis Streams/RabbitMQ) gera overengineering | `TRANSCRICAO` | `[09:07] Diego` |
| `RFC-ALT-03` | `docs/RFC.md` | Alternativa Descartada | Triggers nativas no MySQL não possuem sistema NOTIFY/LISTEN | `TRANSCRICAO` | `[09:09] Diego` |
| `RFC-QUESTAO-01` | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída por cliente a ser observado pós-deploy | `TRANSCRICAO` | `[09:39] Larissa` |
| `RFC-QUESTAO-02` | `docs/RFC.md` | Questão em Aberto | Notificação por e-mail mantida fora da release inicial | `TRANSCRICAO` | `[09:37] Larissa` |
| `RFC-QUESTAO-03` | `docs/RFC.md` | Questão em Aberto | Arquivamento automático de eventos entregues após 30 dias | `TRANSCRICAO` | `[09:08] Diego` |
| `RFC-RISCO-01` | `docs/RFC.md` | Risco e Cronograma | Janela obrigatória de 2 dias úteis de revisão de segurança pela Sofia | `TRANSCRICAO` | `[09:46] Sofia` |
| `RFC-CRONO-01` | `docs/RFC.md` | Cronograma | Projeto estimado em 3 Sprints com revisão de segurança incluída | `TRANSCRICAO` | `[09:46] Larissa` |
| `ADR-001` | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão Arquitetural | Adoção do Padrão Outbox no MySQL para enfileiramento atômico | `TRANSCRICAO` | `[09:06] Diego` |
| `ADR-002` | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão Arquitetural | Worker em processo separado rodando polling a cada 2s | `TRANSCRICAO` | `[09:08] Diego` |
| `ADR-003` | `docs/adrs/ADR-003-retry-e-dead-letter-queue.md` | Decisão Arquitetural | Política de retry 5x backoff exponencial e DLQ | `TRANSCRICAO` | `[09:17] Diego` |
| `ADR-004` | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secrets.md` | Decisão Arquitetural | Autenticação HMAC-SHA256, secret por endpoint e rotação em 24h | `TRANSCRICAO` | `[09:20] Sofia` |
| `ADR-005` | `docs/adrs/ADR-005-entrega-at-least-once-e-idempotencia.md` | Decisão Arquitetural | Entrega At-Least-Once e idempotência via header X-Event-Id | `TRANSCRICAO` | `[09:24] Diego` |
| `FDD-FLUXO-01` | `docs/FDD.md` | Fluxo Detalhado | Snapshot estático JSON do payload renderizado na transação | `TRANSCRICAO` | `[09:52] Larissa` |
| `FDD-FLUXO-02` | `docs/FDD.md` | Fluxo Detalhado | Formato do payload JSON com campos snake_case enxutos sem items | `TRANSCRICAO` | `[09:43] Diego` |
| `FDD-FLUXO-03` | `docs/FDD.md` | Fluxo Detalhado | Filtro de eventos por status na inserção da outbox | `TRANSCRICAO` | `[09:34] Bruno` |
| `FDD-HEADERS-01` | `docs/FDD.md` | Restrição de Interface | Headers obrigatórios: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | `TRANSCRICAO` | `[09:44] Diego` |
| `FDD-TIMEOUT-01` | `docs/FDD.md` | Restrição Técnica | Timeout máximo de requisição HTTP do worker configurado em 10s | `TRANSCRICAO` | `[09:42] Diego` |
| `FDD-LIMIT-01` | `docs/FDD.md` | Limitação Conhecida | Garantia de ordering apenas por order_id em single-worker | `TRANSCRICAO` | `[09:13] Diego` |
| `FDD-RISCO-01` | `docs/FDD.md` | Risco Técnico | Sobrecarga do MySQL por polling contínuo a cada 2s | `TRANSCRICAO` | `[09:08] Diego` |
| `FDD-RISCO-02` | `docs/FDD.md` | Risco Técnico | Bloqueio do worker por endpoint de cliente lento (timeout 10s) | `TRANSCRICAO` | `[09:42] Diego` |
| `FDD-RISCO-03` | `docs/FDD.md` | Risco Técnico | Rejeição em massa durante rotação de secrets (grace period 24h) | `TRANSCRICAO` | `[09:21] Sofia` |
| `FDD-INT-01` | `docs/FDD.md` | Integração de Código | Inserção na outbox no método changeStatus via transaction tx | `CODIGO` | `src/modules/orders/order.service.ts` |
| `FDD-INT-02` | `docs/FDD.md` | Integração de Código | Proteção de rotas CRUD e replay com authenticate e requireRole('ADMIN') | `CODIGO` | `src/middlewares/auth.middleware.ts` |
| `FDD-INT-03` | `docs/FDD.md` | Integração de Código | Exceções de webhook estendendo AppError com prefixo WEBHOOK_* | `CODIGO` | `src/shared/errors/app-error.ts` |
| `FDD-INT-04` | `docs/FDD.md` | Integração de Código | Interceptação centralizada de erros no Express error middleware | `CODIGO` | `src/middlewares/error.middleware.ts` |
| `FDD-INT-05` | `docs/FDD.md` | Integração de Código | Logging estruturado e auditoria com a instância singleton do Pino | `CODIGO` | `src/shared/logger/index.ts` |
| `FDD-INT-06` | `docs/FDD.md` | Integração de Código | Modelos de banco de dados WebhookSubscription, Outbox, DLQ e Logs | `CODIGO` | `prisma/schema.prisma` |

