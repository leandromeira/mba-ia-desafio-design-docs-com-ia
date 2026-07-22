# FDD: Sistema de Webhooks de Notificação de Pedidos

- **Versão**: 1.0.0
- **Data**: 2026-07-22
- **Responsáveis**: Larissa, Diego, Bruno, Sofia, Marcos

---

### 1. Contexto e motivação técnica

O Order Management System (OMS) opera com uma arquitetura REST em Node.js, TypeScript, Express, Prisma ORM e banco MySQL. Atualmente, os clientes B2B (*Atlas Comercial*, *MaxDistribuição*, *Nova Cargo*) consultam periodicamente o endpoint `GET /orders` via *polling* para detectar alterações no status de seus pedidos (`[09:00] Marcos`). Essa abordagem consome recursos excessivos da API, gera latência indesejada e encarece a integração (`[09:00] Marcos`).

A empresa precisa fornecer uma solução de notificações ativas **Outbound** (do OMS para os clientes B2B) que garanta a entrega dos eventos de transição de status de pedidos com latência inferior a **10 segundos** (`[09:00] Marcos`, `[09:02] Marcos`, `[09:03] Sofia`).

O Feature Design Document (FDD) detalha a especificação técnica de implementação do módulo de webhooks, descrevendo os fluxos de persistência atômica, o worker desacoplado, o mecanismo de resiliência e retry, a política de Dead Letter Queue (DLQ), os esquemas de segurança HMAC-SHA256, os contratos de API REST, a matriz de erros padronizada e a integração com os componentes existentes da codebase.

---

### 2. Objetivos técnicos

- **Latência de Entrega**: Garantir que as notificações de webhooks sejam disparadas em menos de 10 segundos após a confirmação da transição de status do pedido, operando com polling a cada 2 segundos (`[09:02] Marcos`, `[09:10] Larissa`).
- **Consistência Atômica Relacional**: Garantir que o enfileiramento do evento ocorra estritamente dentro da transação SQL que altera o pedido em `src/modules/orders/order.service.ts`, eliminando perdas ou eventos fantasmas (`[09:06] Diego`).
- **Isolamento de Processos**: Garantir que o worker de envio seja executado em um processo Node.js independente (`src/worker.ts`), sem impactar o *event loop* da API HTTP principal (`src/server.ts`) (`[09:11] Diego`).
- **Resiliência e Recuperação de Falhas**: Garantir uma janela de retentativas de aproximadamente 15 horas via 5 retries com backoff exponencial (1m, 5m, 30m, 2h, 12h) e isolamento em Dead Letter Queue (DLQ) (`[09:17] Diego`, `[09:18] Diego`).
- **Segurança de Tráfego e Autenticidade**: Garantir a autenticidade dos dados via assinatura HMAC-SHA256, rotação de chaves com *grace period* de 24 horas e protocolo HTTPS obrigatório com validação Zod (`[09:20] Sofia`, `[09:21] Sofia`, `[09:23] Sofia`).

---

### 3. Escopo e exclusões

**Incluído**
- Modelagem das tabelas `webhook_subscriptions`, `webhook_outbox`, `webhook_delivery_logs` e `webhook_dead_letter` no `prisma/schema.prisma` (`[09:06] Diego`, `[09:18] Diego`, `[09:21] Bruno`).
- Função pura de enfileiramento `publishWebhookEvent(tx, ...)` integrada na transação `changeStatus` em `src/modules/orders/order.service.ts` (`[09:41] Bruno`).
- Processo worker isolado em `src/worker.ts` acionado por `npm run worker` executando polling no MySQL a cada 2 segundos (`[09:08] Diego`, `[09:11] Larissa`).
- Mecanismo de retry de 5 tentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h) e transferência para DLQ (`[09:17] Diego`, `[09:18] Diego`).
- Endpoints REST HTTP para CRUD de configurações de webhook do cliente e consulta de histórico de entregas (`[09:31] Marcos`, `[09:34] Marcos`).
- Endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay` protegido pelo middleware `requireRole('ADMIN')` para reprocessamento manual de mensagens da DLQ com audit log (`[09:18] Diego`, `[09:36] Sofia`).
- Assinatura HMAC-SHA256 no header `X-Signature`, secret por endpoint, suporte a rotação com chave anterior válida por 24h, HTTPS obrigatório e teto de payload de 64KB (`[09:20] Sofia`, `[09:21] Sofia`, `[09:23] Sofia`, `[09:24] Diego`).
- Semântica de entrega *At-Least-Once* utilizando o header `X-Event-Id` (UUID v4) para deduplicação no cliente B2B (`[09:24] Diego`, `[09:25] Diego`).

**Excluído**
- Envio de notificações de alerta por e-mail em caso de falhas consecutivas de webhook (`[09:37] Larissa`).
- Interface gráfica ou Dashboard visual no frontend para gerenciamento de webhooks (`[09:40] Larissa`).
- Algoritmo de *Rate Limiting* de saída por cliente no worker (`[09:39] Larissa`).
- Rotina de arquivamento/exclusão automática de registros entregues após 30 dias na outbox (`[09:08] Diego`).
- Garantia de entrega *Exactly-Once* ou chamadas HTTP síncronas no serviço de pedidos (`[09:04] Bruno`, `[09:25] Diego`).

---

### 4. Fluxos detalhados e diagramas

#### Fluxo 1: Enfileiramento na Outbox (Produtor de Eventos)
1. A API HTTP recebe uma requisição `PATCH /orders/:id/status` autenticada.
2. O controlador aciona o método `OrderService.changeStatus` em `src/modules/orders/order.service.ts`.
3. O serviço inicia uma transação atômica no banco de dados via `this.prisma.$transaction(async (tx) => { ... })`.
4. Dentro do bloco transacional `tx`:
   a. Valida a transição de estado via `canTransition(from, to)`.
   b. Atualiza ou debita o estoque de produtos em `products`.
   c. Atualiza o status do pedido na tabela `orders`.
   d. Cria o registro de auditoria na tabela `order_status_history`.
   e. Busca as configurações de webhooks ativas na tabela `webhook_subscriptions` para o `customerId` que possuem interesse no novo status (`[09:33] Bruno`, `[09:34] Bruno`).
   f. Caso existam subscrições ativas, renderiza o snapshot JSON estático do payload (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`) (`[09:43] Diego`, `[09:52] Larissa`).
   g. Executa o `INSERT` na tabela `webhook_outbox` com `status = 'PENDING'`, `attempts = 0` e `eventId = UUIDv4` (`[09:06] Diego`, `[09:25] Diego`, `[09:51] Larissa`).
5. A transação SQL é commitada. Se qualquer passo falhar, toda a operação sofre `rollback` e nenhum evento é enfileirado (`[09:06] Diego`).

#### Fluxo 2: Processamento e Disparo de Notificações (Worker)
1. O processo `src/worker.ts` roda continuamente em loop com pausa de 2 segundos entre ciclos (`[09:08] Diego`, `[09:09] Diego`).
2. A cada ciclo, o worker consulta no MySQL os registros da `webhook_outbox` ordenados por `created_at ASC` onde `status = 'PENDING'` ou (`status = 'FAILED'` e `next_retry_at <= NOW()`) (`[09:08] Diego`).
3. Para cada evento localizado:
   a. Carrega a *secret* ativa da assinatura correspondente em `webhook_subscriptions`.
   b. Calcula a assinatura HMAC-SHA256 sobre a string estática do payload JSON (`[09:20] Sofia`).
   c. Prepara os headers da requisição: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` (`[09:44] Diego`, `[09:44] Sofia`).
   d. Dispara uma requisição HTTP `POST` para a URL do webhook com timeout configurado de 10 segundos (`[09:42] Diego`).
4. Se o endpoint do cliente responder com código de status HTTP `2xx`:
   a. Atualiza o status do evento na `webhook_outbox` para `DELIVERED`.
   b. Grava um registro de log de entrega em `webhook_delivery_logs` com o código de resposta HTTP, tempo de resposta em milissegundos e timestamp (`[09:34] Marcos`).

#### Fluxo 3: Retry com Backoff Exponencial e Transferência para DLQ
1. Se a chamada HTTP falhar por erro de rede, recusa de conexão, status HTTP != 2xx ou exceder o timeout de 10 segundos (`[09:42] Diego`):
   a. Incrementa o contador `attempts` do evento em `webhook_outbox`.
   b. Grava uma entrada de falha no histórico em `webhook_delivery_logs`.
2. Se `attempts < 5`:
   a. Calcula a data da próxima tentativa (`next_retry_at`) adicionando o intervalo da progressão exponencial (`[09:17] Diego`):
      - 1ª Falha: 1 minuto
      - 2ª Falha: 5 minutos
      - 3ª Falha: 30 minutos
      - 4ª Falha: 2 horas
      - 5ª Falha: 12 horas
   b. Atualiza o status do evento na `webhook_outbox` para `FAILED` e salva `next_retry_at`.
3. Se `attempts >= 5`:
   a. O evento atingiu a falha permanente após 15 horas de tentativas (`[09:17] Diego`).
   b. Move o evento da `webhook_outbox` para a tabela `webhook_dead_letter`, gravando o payload completo, o último erro capturado e a data de esgotamento (`[09:18] Diego`).
   c. Marca o registro na `webhook_outbox` com o status `DEAD_LETTER`.

#### Fluxo 4: Replay Manual de Mensagens da DLQ (Administrativo)
1. Um usuário autenticado com a *role* `ADMIN` dispara uma requisição `POST /admin/webhooks/dead-letter/:id/replay` (`[09:18] Diego`, `[09:35] Larissa`).
2. O middleware `authenticate` extrai o token JWT e o middleware `requireRole('ADMIN')` valida se o usuário possui permissão de administrador (`[09:36] Sofia`).
3. O serviço de webhook localiza o registro na tabela `webhook_dead_letter`:
   a. Reinsere o evento na tabela `webhook_outbox` com `status = 'PENDING'`, `attempts = 0` e `next_retry_at = NOW()`.
   b. Atualiza a entrada na DLQ com indicação de que o evento foi reprocessado.
4. Grava um log de auditoria no Pino logger (`src/shared/logger/index.ts`) identificando o `userId` do administrador que executou a ação e o ID da DLQ (`[09:36] Sofia`).

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

#### Contrato 1: Cadastrar Webhook (`POST /webhooks`)
- **Tipo**: `http_endpoint`
- **Rota**: `POST /webhooks`
- **Autenticação**: Requer JWT (Roles: `ADMIN` ou `OPERATOR`) (`[09:36] Sofia`)
- **Semântica**: Cadastra um novo endpoint de webhook para receber notificações de mudanças de status de pedidos. A *secret* é gerada aleatoriamente pelo OMS e retornada no payload de resposta (`[09:31] Marcos`).

**Exemplo de requisição**
```json
{
  "customerId": "c1f8a9b2-3d4e-4f5a-6b7c-8d9e0f1a2b3c",
  "url": "https://api.atlascomercial.com.br/v1/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Exemplo de resposta (HTTP 201 Created)**
```json
{
  "id": "7b8a9c0d-1e2f-4a3b-8c6d-7e8f9a0b1c2d",
  "customerId": "c1f8a9b2-3d4e-4f5a-6b7c-8d9e0f1a2b3c",
  "url": "https://api.atlascomercial.com.br/v1/webhooks/orders",
  "secret": "whsec_9k8j7h6g5f4d3s2a1q0w9e8r7t6y5u4i3o2p1l0k",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-07-22T10:00:00.000Z"
}
```

---

#### Contrato 2: Listar Webhooks do Cliente (`GET /webhooks`)
- **Tipo**: `http_endpoint`
- **Rota**: `GET /webhooks?customerId=:customerId`
- **Autenticação**: Requer JWT (Roles: `ADMIN` ou `OPERATOR`)
- **Semântica**: Retorna a lista de webhooks cadastrados para o cliente informado (`[09:33] Bruno`).

**Exemplo de resposta (HTTP 200 OK)**
```json
{
  "data": [
    {
      "id": "7b8a9c0d-1e2f-4a3b-8c6d-7e8f9a0b1c2d",
      "customerId": "c1f8a9b2-3d4e-4f5a-6b7c-8d9e0f1a2b3c",
      "url": "https://api.atlascomercial.com.br/v1/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-22T10:00:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

---

#### Contrato 3: Rotacionar Secret do Webhook (`POST /webhooks/:id/rotate-secret`)
- **Tipo**: `http_endpoint`
- **Rota**: `POST /webhooks/:id/rotate-secret`
- **Autenticação**: Requer JWT (Roles: `ADMIN` ou `OPERATOR`)
- **Semântica**: Gera uma nova *secret* para o webhook. A *secret* anterior permanece válida pelo período de transição (*grace period*) de 24 horas no campo `previousSecretExpiresAt` (`[09:21] Sofia`).

**Exemplo de resposta (HTTP 200 OK)**
```json
{
  "id": "7b8a9c0d-1e2f-4a3b-8c6d-7e8f9a0b1c2d",
  "newSecret": "whsec_NEW_1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p",
  "previousSecretExpiresAt": "2026-07-23T10:00:00.000Z",
  "message": "Nova secret gerada. A secret anterior continuará válida por 24 horas para transição."
}
```

---

#### Contrato 4: Consultar Histórico de Entregas (`GET /webhooks/:id/deliveries`)
- **Tipo**: `http_endpoint`
- **Rota**: `GET /webhooks/:id/deliveries`
- **Autenticação**: Requer JWT (Roles: `ADMIN` ou `OPERATOR`)
- **Semântica**: Retorna o histórico das últimas tentativas de entrega de webhooks efetuadas pelo worker (`[09:34] Marcos`).

**Exemplo de resposta (HTTP 200 OK)**
```json
{
  "data": [
    {
      "id": "1a2b3c4d-5e6f-4a7b-8c9d-1e2f3a4b5c6d",
      "webhookId": "7b8a9c0d-1e2f-4a3b-8c6d-7e8f9a0b1c2d",
      "eventId": "9f8e7d6c-5b4a-4f2e-8d0c-9b8a7f6e5d4c",
      "eventType": "order.status_changed",
      "statusCode": 200,
      "responseTimeMs": 145,
      "success": true,
      "attempt": 1,
      "deliveredAt": "2026-07-22T10:05:02.145Z"
    }
  ],
  "page": 1,
  "pageSize": 50,
  "total": 1
}
```

---

#### Contrato 5: Replay Manual de Mensagem da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`)
- **Tipo**: `http_endpoint`
- **Rota**: `POST /admin/webhooks/dead-letter/:id/replay`
- **Autenticação**: Requer JWT com Role estrita **`ADMIN`** (`[09:35] Larissa`, `[09:36] Sofia`)
- **Semântica**: Reinsere uma mensagem mantida na Dead Letter Queue de volta na outbox com status `PENDING` para reprocessamento pelo worker (`[09:18] Diego`). Auditado no log.

**Exemplo de resposta (HTTP 200 OK)**
```json
{
  "deadLetterId": "5a4b3c2d-1e0f-4a8b-8c6d-5e4f3a2b1c0d",
  "outboxId": "0a9b8c7d-6e5f-4a3b-8c1d-0e9f8a7b6c5d",
  "status": "REENFILEIRADO",
  "replayedAt": "2026-07-22T10:10:00.000Z",
  "replayedByUserId": "a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d"
}
```

---

#### Contrato do Payload Notificado pelo Worker ao Cliente B2B
- **Método HTTP**: `POST`
- **Headers Enviados**:
  - `Content-Type`: `application/json`
  - `X-Event-Id`: `9f8e7d6c-5b4a-4f2e-8d0c-9b8a7f6e5d4c` (UUID v4 do evento) (`[09:25] Diego`)
  - `X-Signature`: `sha256=a8f5f167f44f4964e6c998dee827110c...` (HMAC-SHA256) (`[09:20] Sofia`)
  - `X-Timestamp`: `1784714702` (Epoch Unix timestamp em segundos) (`[09:44] Diego`)
  - `X-Webhook-Id`: `7b8a9c0d-1e2f-4a3b-8c6d-7e8f9a0b1c2d` (`[09:44] Sofia`)

**Exemplo de Payload JSON Enviado (Teto de tamanho: 64KB)** (`[09:24] Diego`, `[09:43] Diego`)
```json
{
  "event_id": "9f8e7d6c-5b4a-4f2e-8d0c-9b8a7f6e5d4c",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-22T10:05:00.000Z",
  "order_id": "8f7e6d5c-4b3a-4f1e-8d9c-8b7a6f5e4d3c",
  "order_number": "ORD-2026-00123",
  "customer_id": "c1f8a9b2-3d4e-4f5a-6b7c-8d9e0f1a2b3c",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 15990
}
```

---

### 6. Erros, exceções e fallback

- **Matriz de Erros Previstos (Códigos Padronizados com Prefixo `WEBHOOK_*`)**:

| Código de Erro | Status HTTP | Condição / Causa | Tratamento / Resposta |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 Not Found | ID da assinatura de webhook ou da mensagem DLQ não existe | Retorna JSON de erro estendido de `AppError` |
| `WEBHOOK_INVALID_URL` | 400 Bad Request | URL informada não usa protocolo HTTPS (`https://`) | Recusado no middleware Zod `validate.middleware.ts` (`[09:23] Sofia`) |
| `WEBHOOK_SECRET_REQUIRED` | 400 Bad Request | Tentativa de operação sem chave válida | Recusado com mensagem amigável |
| `WEBHOOK_MAX_PAYLOAD_EXCEEDED` | 422 Unprocessable | Snapshot do payload excede o limite máximo de 64KB | Inserção cancelada com log de erro (`[09:23] Sofia`) |
| `WEBHOOK_DELIVERY_FAILED` | 502 Bad Gateway | Endpoint do cliente B2B retornou status HTTP != 2xx ou timeout | Worker registra falha, incrementa retry e aplica backoff |
| `WEBHOOK_UNAUTHORIZED_REPLAY` | 403 Forbidden | Usuário sem role `ADMIN` tentou executar o replay de DLQ | Interceptado pelo `requireRole('ADMIN')` (`[09:36] Sofia`) |

- **Estratégias de Resiliência**:
  - **Timeout de Conexão**: Cada chamada HTTP realizada pelo worker tem um tempo máximo de espera de 10 segundos (`[09:42] Diego`).
  - **Backoff Exponencial**: 5 tentativas com intervalos progressivos (1m, 5m, 30m, 2h, 12h) (`[09:17] Diego`).
  - **Isolamento em DLQ**: Falhas permanentes não travam novos eventos pendentes.
- **Invariantes Técnicos**:
  - Evento de webhook só existe se a alteração do status do pedido for confirmada na mesma transação SQL (`[09:06] Diego`).
  - Assinatura HMAC é estritamente recalculada no worker utilizando o corpo original da requisição (`[09:20] Sofia`).

---

### 7. Observabilidade

#### Logs Estruturados (Pino Logger)
- Reuso da instância singleton do Pino Logger já configurada no projeto em `src/shared/logger/index.ts` (`[09:29] Bruno`, `[09:30] Larissa`).
- Emissão de logs estruturados em formato JSON durante as rotinas de polling e tentativas de entrega executadas pelo worker em `src/worker.ts` (`[09:29] Bruno`).
- **Logs de Auditoria**: Registro obrigatório de auditoria capturando a identificação do usuário (`userId` com a role `ADMIN`) ao executar o reprocessamento manual de mensagens da DLQ via `POST /admin/webhooks/dead-letter/:id/replay` (`[09:36] Sofia`).

---

### 8. Dependências e compatibilidade

| Componente / Biblioteca | Versão Mínima | Finalidade no Módulo de Webhooks |
| --- | --- | --- |
| **Node.js** | `>= 20.0.0` | Runtime de execução da API principal (`src/server.ts`) e do Worker (`src/worker.ts`) |
| **TypeScript** | `5.6.3` | Tipagem estrita de contratos, repositórios e schemas DTO |
| **Express** | `4.21.1` | Roteamento dos endpoints REST de gerenciamento de webhooks |
| **Prisma ORM** | `5.22.0` | Mapeamento das tabelas de webhook no MySQL e controle transacional (`$transaction`) |
| **Zod** | `3.23.8` | Validação de schemas de entrada e obrigatoriedade de HTTPS |
| **Pino** | `9.5.0` | Instância centralizada de logging estruturado |
| **UUID** | `11.0.3` | Geração de identificadores únicos UUID v4 para o `X-Event-Id` |

---

### 9. Integração com o sistema existente

Para garantir a coerência arquitetural e reutilizar os padrões da aplicação, o módulo de webhooks se integra com **6 caminhos de arquivos reais** da codebase:

1. **`src/modules/orders/order.service.ts`**:
   - O método `changeStatus` executa as atualizações dentro do bloco `this.prisma.$transaction(async (tx) => { ... })`.
   - A integração estende este método para invocar a função de enfileiramento `publishWebhookEvent(tx, order, fromStatus, toStatus)`, passando o cliente de transação `tx` para gravar o evento na tabela `webhook_outbox` de forma atômica na mesma transação SQL (`[09:41] Bruno`).

2. **`src/middlewares/auth.middleware.ts`**:
   - Reutiliza os middlewares de autenticação `authenticate` e de autorização `requireRole`.
   - Os endpoints de gerenciamento de webhooks (`/webhooks`) utilizam `authenticate`. O endpoint administrativo de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) utiliza obrigatoriamente `requireRole('ADMIN')` (`[09:36] Sofia`).

3. **`src/shared/errors/app-error.ts` e `src/shared/errors/index.ts`**:
   - Todas as exceções do módulo de webhooks estendem a classe base `AppError`.
   - Reutiliza a convenção de códigos de erro padronizados da aplicação, utilizando exclusivamente o prefixo `WEBHOOK_*` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_MAX_PAYLOAD_EXCEEDED`) (`[09:28] Bruno`, `[09:29] Larissa`).

4. **`src/middlewares/error.middleware.ts`**:
   - O middleware de tratamento centralizado de erros intercepta automaticamente as exceções `AppError` do módulo de webhooks, erros do Zod (`ZodError`) e exceções do Prisma, retornando respostas HTTP padronizadas sem necessidade de alterar o middleware global (`[09:29] Bruno`).

5. **`src/shared/logger/index.ts`**:
   - Reutiliza a instância singleton do Pino Logger para registrar o início dos ciclos de polling, tentativas de envio HTTP, falhas e auditoria dos comandos de replay executados por administradores (`[09:29] Bruno`).

6. **`prisma/schema.prisma`**:
   - Estende o arquivo de modelos do Prisma criando as novas entidades relacionais (`WebhookSubscription`, `WebhookOutbox`, `WebhookDeliveryLog`, `WebhookDeadLetter`) utilizando UUID (v4) `@db.Char(36)` como chave primária, alinhando-se ao padrão de todas as tabelas do projeto (`[09:51] Larissa`).

---

### 10. Critérios de aceite técnicos

- ☐ **Atomicidade**: Inserção de eventos na outbox ocorre dentro da transação de `changeStatus` em `src/modules/orders/order.service.ts`; se a mudança de status do pedido sofrer rollback, a outbox também sofre rollback.
- ☐ **Isolamento de Processo**: O worker é executado em processo separado (`src/worker.ts`) por `npm run worker`, estabelecendo sua própria instância do `PrismaClient`.
- ☐ **Polling e Latência**: Worker consulta a tabela `webhook_outbox` a cada 2 segundos, entregando os eventos pendentes com latência total inferior a 10 segundos no pior caso.
- ☐ **Retry e DLQ**: Eventos com falha são reprocessados em 5 tentativas com backoff exponencial (1m, 5m, 30m, 2h, 12h). Na 5ª falha, o evento é transferido para a tabela `webhook_dead_letter`.
- ☐ **Replay Admin**: Endpoint `POST /admin/webhooks/dead-letter/:id/replay` reinsere a mensagem na outbox e exige obrigatoriamente a role `ADMIN`.
- ☐ **Segurança HMAC**: Header `X-Signature` enviado com o hash HMAC-SHA256 do payload. URLs `http://` são recusadas pelo Zod com o erro `WEBHOOK_INVALID_URL`. Rotação de secret mantém a chave antiga ativa por 24h.
- ☐ **Idempotência**: Header `X-Event-Id` enviado com UUID v4 único por evento.
- ☐ **Integração no Código**: Código reutiliza `AppError` com prefixo `WEBHOOK_*`, Pino logger em `src/shared/logger/index.ts` e error middleware.

---

### 11. Riscos e mitigação

#### Risco 1: Sobrecarga do MySQL por Polling Contínuo a Cada 2s
- **Probabilidade**: Média
- **Impacto**: Médio
- **Mitigação**: Criação de índice composto em `[status, created_at]` na tabela `webhook_outbox` no `prisma/schema.prisma`. As consultas de polling leem apenas índices cobertos com limite de batch pequeno (`[09:08] Diego`).
- **Plano de contingência**: Manter a leitura de eventos pendentes restrita a lotes (batch) pequenos para limitar a carga de I/O no banco (`[09:08] Diego`).

#### Risco 2: Bloqueio do Worker por Indisponibilidade de Endpoint de Cliente Lento
- **Probabilidade**: Alta
- **Impacto**: Médio
- **Mitigação**: Timeout rígido de 10 segundos em todas as requisições HTTP disparadas pelo worker (`[09:42] Diego`).
- **Plano de contingência**: Se o endpoint do cliente exceder 10s, o envio é abortado, marcado como falha e reagendado para o próximo ciclo de retry.

#### Risco 3: Rejeição em Massa de Webhooks durante Rotação de Secrets pelos Clientes B2B
- **Probabilidade**: Média
- **Impacto**: Alto
- **Mitigação**: Implementação de *grace period* de 24 horas (`previousSecretExpiresAt`), permitindo a validação paralela pela chave antiga enquanto o cliente atualiza seus sistemas (`[09:21] Sofia`).
- **Plano de contingência**: Manter a chave anterior válida durante o período transitório estrito de 24 horas para permitir a atualização nos sistemas do cliente (`[09:21] Sofia`).
