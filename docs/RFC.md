# RFC: Sistema de Webhooks de Notificação de Pedidos

- **Autor**: Equipe de Engenharia OMS
- **Status**: Em Revisão (In Review)
- **Data**: 2026-07-22
- **Revisores**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Resumo Executivo (TL;DR)

Este documento propõe a arquitetura para o novo **Sistema de Webhooks de Notificação de Pedidos** do Order Management System (OMS). A solução tem como objetivo notificar clientes B2B (*Atlas Comercial*, *MaxDistribuição*, *Nova Cargo*) em tempo quase real (latência < 10 segundos) sempre que o status de um pedido for alterado na plataforma (`[09:00] Marcos`, `[09:02] Marcos`).

A proposta baseia-se no **Padrão Outbox no MySQL**, onde o evento de notificação é gravado de forma atômica dentro da mesma transação SQL da mudança de status do pedido (`[09:06] Diego`). Um **Worker Node.js em processo separado** (`src/worker.ts`) realiza polling a cada 2 segundos na tabela `webhook_outbox`, efetua os disparos HTTP com timeout de 10 segundos e gerencia uma política de **5 tentativas de retry com backoff exponencial** (1m, 5m, 30m, 2h, 12h) (`[09:08] Diego`, `[09:11] Diego`, `[09:17] Diego`).

Eventos que esgotarem as tentativas são movidos para a **Dead Letter Queue (DLQ)** (`webhook_dead_letter`), com reprocessamento manual via endpoint administrativo restrito à role `ADMIN` (`[09:18] Diego`, `[09:36] Sofia`). A segurança é garantida por assinaturas **HMAC-SHA256** no header `X-Signature`, *secrets* únicas por endpoint com suporte a rotação com *grace period* de 24 horas, HTTPS obrigatório e limite de payload de 64KB (`[09:20] Sofia`, `[09:21] Sofia`, `[09:23] Sofia`, `[09:24] Diego`). A semântica de entrega é **At-Least-Once**, utilizando o header `X-Event-Id` (UUID v4) para controle de idempotência e deduplicação no cliente B2B (`[09:24] Diego`, `[09:25] Diego`).

---

## Contexto e Problema

Atualmente, os clientes B2B dependem de requisições de *polling* contínuo no endpoint `GET /orders` para verificar se houve alteração no status de seus pedidos (`[09:00] Marcos`). Essa abordagem gera alto tráfego ineficiente na API, aumenta os custos de infraestrutura e causa lentidão nas integrações dos clientes (`[09:00] Marcos`).

O cliente *Atlas Comercial* sinalizou risco iminente de migração para um concorrente caso um mecanismo de notificação ativa em tempo real não seja disponibilizado até o fim do trimestre (`[09:00] Marcos`). A latência máxima aceitável pelos clientes para definir "tempo real" foi estabelecida em **10 segundos** (`[09:02] Marcos`).

O escopo da solução engloba exclusivamente notificações **Outbound** (do OMS para os sistemas dos clientes B2B) (`[09:02] Sofia`, `[09:03] Sofia`). A aplicação atual não possui infraestrutura de mensageria ou enfileiramento de eventos, exigindo uma solução resiliente, segura e integrada à stack Node.js/TypeScript e banco MySQL existentes (`[09:07] Diego`, `[09:30] Bruno`).

---

## Proposta Técnica

A arquitetura proposta está dividida em 5 pilares fundamentais, garantindo alto desacoplamento, resiliência e integridade relacional:

### 1. Enfileiramento Atômico (Outbox Pattern)
- Na alteração de status do pedido em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), a gravação do evento de notificação é realizada via `INSERT` na tabela `webhook_outbox` dentro da mesma transação SQL (`this.prisma.$transaction`) (`[09:06] Diego`).
- O payload do evento é gravado como um snapshot estático em formato JSON no momento do commit da transação, garantindo a fidelidade do estado no momento do evento (`[09:52] Larissa`, `[09:52] Diego`).

### 2. Processo Worker Desacoplado
- O processamento de envios é executado por um worker isolado em `src/worker.ts` (iniciado via `npm run worker`), desacoplado do processo da API HTTP principal (`src/server.ts`) (`[09:11] Larissa`, `[09:11] Diego`).
- O worker estabelece sua própria instância do `PrismaClient` e realiza polling a cada 2 segundos buscando registros com status `PENDING` ordenados por `created_at` (`[09:08] Diego`, `[09:30] Bruno`).

### 3. Resiliência, Retry e DLQ
- Cada chamada HTTP de webhook enviada pelo worker possui um timeout máximo de 10 segundos (`[09:42] Diego`).
- Em caso de falha de conexão ou timeout, o evento é remarcado para retentativa seguindo o backoff exponencial de **1m, 5m, 30m, 2h e 12h** (cobrindo ~15 horas) (`[09:17] Diego`).
- Após 5 falhas consecutivas, o evento é transferido para a tabela `webhook_dead_letter` (`[09:18] Diego`).
- Replay manual de mensagens da DLQ é disponibilizado via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, que exige a role `ADMIN` (via `requireRole('ADMIN')` em `src/middlewares/auth.middleware.ts`) e gera log de auditoria no Pino (`[09:18] Diego`, `[09:36] Sofia`).

### 4. Segurança e Assinatura de Payload
- Assinatura no header `X-Signature` calculada via HMAC-SHA256 sobre o corpo JSON da requisição com a *secret* do endpoint (`[09:20] Sofia`).
- *Secret* única por endpoint de webhook cadastrado (`[09:21] Sofia`).
- Suporte a rotação de secret com *grace period* de 24 horas (`previousSecretExpiresAt`), permitindo que mensagens continuem sendo validadas pela chave antiga durante a transição (`[09:21] Sofia`).
- Protocolo HTTPS (`https://`) obrigatório na URL do webhook, validado via schema Zod seguindo o padrão existente em `src/middlewares/validate.middleware.ts` (`[09:23] Sofia`).
- Limite máximo de payload de 64KB; eventos que excederem o limite são rejeitados com erro `WEBHOOK_MAX_PAYLOAD_EXCEEDED` (`[09:23] Sofia`, `[09:24] Diego`).

### 5. Semântica de Entrega e Idempotência
- Garantia de entrega **At-Least-Once** (`[09:24] Diego`).
- Envio do header `X-Event-Id` contendo um UUID v4 único por evento (gerado na inserção da outbox) para permitir que o cliente B2B realize deduplicação do seu lado (`[09:25] Diego`, `[09:26] Larissa`).
- Headers padrão transmitidos em cada request: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` (`[09:44] Diego`, `[09:44] Sofia`).

---

## Alternativas Consideradas

### 1. Disparo HTTP Síncrono no `OrderService` (`[09:03] Larissa`, `[09:04] Bruno`)
- **Descrição**: Executar a chamada HTTP para o endpoint do cliente B2B diretamente dentro do método `changeStatus` em `src/modules/orders/order.service.ts`.
- **Trade-off e Motivo do Descarte**: Se o cliente B2B estivesse lento ou fora do ar, a requisição da API principal travaria e causaria o *rollback* indevido da alteração de status do pedido (`[09:04] Bruno`). Descartado por colocar em risco a estabilidade do OMS (`[09:06] Diego`).

### 2. Mensageria Dedicada Externa (Redis Streams / RabbitMQ) (`[09:07] Larissa`, `[09:07] Diego`)
- **Descrição**: Adotar um cluster de Redis Streams ou RabbitMQ para enfileirar as notificações de webhooks.
- **Trade-off e Motivo do Descarte**: Exigiria provisionar, monitorar e manter um novo cluster de infraestrutura para uma equipe pequena, gerando *overengineering* (`[09:07] Diego`). Descartado pois o banco MySQL existente resolve a demanda com menor complexidade (`[09:07] Diego`).

### 3. Triggers Nativas da Tabela `orders` no MySQL (`[09:09] Bruno`, `[09:09] Diego`)
- **Descrição**: Criar triggers no MySQL para disparar notificações quando o status do pedido fosse alterado.
- **Trade-off e Motivo do Descarte**: O MySQL não possui mecanismo nativo de pub/sub para processos externos tipo `NOTIFY/LISTEN` do PostgreSQL (`[09:09] Diego`). Descartado por inviabilidade técnica de notificação reativa no MySQL (`[09:09] Diego`).

---

## Questões em Aberto

### 1. Rate Limiting de Saída no Worker por Cliente B2B (`[09:38] Diego`, `[09:39] Larissa`)
- **Situação**: Em picos de atualização (ex: 50 pedidos alterados em 1 minuto para o mesmo cliente), o worker disparará 50 requisições simultâneas. 
- **Encaminhamento**: Decidido não implementar rate limiting na release inicial, observando a volumetria em produção antes de adotar travas de envio (`[09:39] Larissa`).

### 2. Notificação por E-mail em Falhas Consecutivas de Webhook (`[09:37] Marcos`, `[09:37] Larissa`)
- **Situação**: Envio de e-mail de alerta para o administrador do cliente B2B quando o webhook falhar 3 vezes seguidas.
- **Encaminhamento**: Declarado fora de escopo para esta fase inicial, postergado para fases futuras pós-mensuração de impacto (`[09:37] Larissa`, `[09:38] Marcos`).

### 3. Arquivamento e Limpeza Automática de Eventos da Outbox (`[09:08] Diego`)
- **Situação**: Exclusão ou arquivamento automático de registros com status `DELIVERED` após 30 dias na tabela `webhook_outbox`.
- **Encaminhamento**: Mapeado como necessidade futura de manutenção do banco, porém fora do escopo de entrega desta release (`[09:08] Diego`).

---

## Impacto e Riscos

- **Impacto em Infraestrutura e Banco**: Aumento na quantidade de operações de escrita no MySQL devido a inserções atômicas em `webhook_outbox` (`[09:07] Bruno`). Mitigado por índices otimizados em `[status, created_at]`.
- **Risco de Segurança em Rotas de Administrador**: O endpoint de replay de mensagens da DLQ exige restrição estrita de acesso. Mitigado com o reuso obrigatório do middleware `requireRole('ADMIN')` e log de auditoria no Pino (`[09:36] Sofia`).
- **Risco de Indisponibilidade na Rotação de Chaves**: Troca abrupta de *secret* poderia causar rejeição de mensagens nos clientes B2B. Mitigado pela implementação do *grace period* de 24 horas (`[09:21] Sofia`).
- **Estimativa de Cronograma**: Projeto estimado em **3 Sprints** de desenvolvimento, incluindo janela final obrigatória de 2 dias úteis reservada para revisão de código pela engenheira de segurança Sofia antes do deploy (`[09:46] Larissa`, `[09:46] Sofia`).

---

## Decisões Relacionadas

- [ADR-001: Adoção do Padrão Outbox no MySQL para Enfileiramento de Webhooks](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em Processo Separado em Polling a Cada 2 Segundos](./adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)](./adrs/ADR-003-retry-e-dead-letter-queue.md)
- [ADR-004: Autenticação via HMAC-SHA256, Secret por Endpoint e Rotação com Grace Period](./adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secrets.md)
- [ADR-005: Garantia de Entrega At-Least-Once e Idempotência via Header X-Event-Id](./adrs/ADR-005-entrega-at-least-once-e-idempotencia.md)
