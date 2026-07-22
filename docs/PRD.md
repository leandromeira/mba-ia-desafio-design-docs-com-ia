# PRD: Sistema de Webhooks de Notificação de Pedidos

- **Versão**: 1.0.0
- **Data**: 2026-07-22
- **Responsáveis**: Larissa, Diego, Bruno, Sofia, Marcos

---

### Resumo

O **Sistema de Webhooks de Notificação de Pedidos** é uma nova funcionalidade do Order Management System (OMS) projetada para notificar de forma ativa (outbound) e em tempo quase real os clientes B2B da plataforma sempre que o status de um pedido for alterado (`[09:00] Marcos`, `[09:03] Sofia`). A solução substitui o modelo ineficiente de consulta por *polling* contínuo no endpoint `GET /orders`, garantindo a entrega dos eventos com latência inferior a 10 segundos via arquivamento atômico (Outbox Pattern no MySQL), processamento por worker isolado e autenticação HMAC-SHA256 (`[09:00] Marcos`, `[09:02] Marcos`, `[09:06] Diego`, `[09:20] Sofia`).

---

### Contexto e problema

**Público-alvo**
- **Clientes B2B da plataforma**: Empresas parceiras (*Atlas Comercial*, *MaxDistribuição*, *Nova Cargo*) que integram seus sistemas de gestão ao OMS (`[09:00] Marcos`).
- **Equipes de Engenharia/Integração B2B**: Desenvolvedores dos clientes responsáveis por consumir e processar os eventos notificados.

**Cenários de uso chave**
- **Notificação Automática de Mudança de Status**: O cliente B2B recebe uma requisição HTTP `POST` em seu servidor assim que seu pedido transita de estado (ex: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) (`[09:00] Marcos`, `[09:33] Marcos`).
- **Recuperação Transparente de Indisponibilidades**: O sistema realiza retentativas automáticas espaçadas caso o endpoint do cliente esteja temporariamente offline ou lento (`[09:15] Diego`, `[09:17] Diego`).
- **Rotação Segura de Chaves**: O cliente B2B rotaciona sua chave secreta via API sem perder notificações durante a transição (`[09:21] Sofia`).

**Onde essa feature será implantada**
- Aplicação Node.js/TypeScript funcional (OMS) já existente em produção, integrada aos módulos de pedidos (`src/modules/orders/`) e banco de dados MySQL via Prisma ORM (`prisma/schema.prisma`).

**Problemas priorizados**
- **Integração Lenta e Custosa**: Os clientes B2B hoje realizam *polling* constante no endpoint `GET /orders` para identificar se houve alterações nos pedidos, sobrecarregando a infraestrutura da API (`[09:00] Marcos`).
- **Risco Iminente de Churn**: O cliente *Atlas Comercial* sinalizou que migrará para um concorrente caso o sistema de notificações em tempo real não seja entregue até o final do trimestre (`[09:00] Marcos`).
- **Falta de Notificação Ativa Outbound**: A aplicação OMS atual não possui qualquer mecanismo de eventos, filas ou notificações para sistemas externos (`[09:02] Sofia`, `[09:03] Sofia`).

---

### Objetivos e métricas

| Objetivo | Métrica | Meta Quantitativa |
| --- | --- | --- |
| Notificação de mudança de status de pedidos em tempo quase real | Latência total de entrega (do commit no banco até o envio HTTP) | **Menos de 10 segundos** (`[09:02] Marcos`, `[09:10] Larissa`) |
| Eliminação do polling ineficiente de pedidos | Volume de chamadas `GET /orders` originadas por clientes B2B | Eliminação de requisições de polling contínuas (`[09:00] Marcos`) |
| Retenção de clientes B2B estratégicos | Taxa de cancelamento (*churn*) motivado por deficiência de integração | **0% de churn** na base B2B solicitante (*Atlas*, *Max*, *Nova Cargo*) (`[09:00] Marcos`) |

---

### Escopo

**Incluso**
- Gravamento atômico de eventos na tabela `webhook_outbox` dentro da mesma transação SQL (`$transaction`) da mudança de status em `OrderService.changeStatus` (`[09:06] Diego`).
- Worker Node.js em processo separado (`src/worker.ts`) acionado por `npm run worker` executando polling no MySQL a cada 2 segundos (`[09:08] Diego`, `[09:11] Larissa`).
- Mecanismo de resiliência com 5 tentativas de retry e progressão de backoff exponencial (1m, 5m, 30m, 2h, 12h) cobrindo ~15 horas (`[09:17] Diego`).
- Transferência automática para Dead Letter Queue em tabela `webhook_dead_letter` após 5 falhas consecutivas (`[09:18] Diego`).
- Endpoint de replay manual de mensagens da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) restrito a usuários com role `ADMIN` (`[09:18] Diego`, `[09:36] Sofia`).
- Assinatura no header `X-Signature` via HMAC-SHA256, *secret* única por endpoint, suporte a rotação com *grace period* de 24 horas (`previousSecretExpiresAt`), protocolo HTTPS obrigatório no schema Zod e limite de payload de 64KB (`[09:20] Sofia`, `[09:21] Sofia`, `[09:23] Sofia`, `[09:24] Diego`).
- Semântica de entrega *At-Least-Once* utilizando o header `X-Event-Id` (UUID v4) para deduplicação no cliente B2B (`[09:24] Diego`, `[09:25] Diego`).
- Endpoints REST HTTP para CRUD de configurações de webhook e histórico de entregas (`[09:31] Marcos`, `[09:34] Marcos`).

**Fora de escopo**
- Envio de e-mails de alerta para o cliente em caso de falhas consecutivas de webhook (`[09:37] Larissa`).
- Painel visual ou Dashboard de interface gráfica para gerenciamento de webhooks (`[09:40] Larissa`).
- Algoritmo de *Rate Limiting* de saída por cliente no worker (`[09:39] Larissa`).
- Rotina de arquivamento/exclusão automática de registros entregues na outbox após 30 dias (`[09:08] Diego`).
- Garantia de entrega *Exactly-Once* ou chamadas síncronas HTTP durante a mudança de status (`[09:04] Bruno`, `[09:25] Diego`).

---

### Requisitos funcionais

#### [FR-001] Cadastro e Configuração de Endpoints de Webhook
- **Descrição**: Permite que a equipe cadastre um novo endpoint HTTP para receber notificações de mudanças de status de pedidos de um determinado cliente B2B (`[09:31] Marcos`).
- **Fluxo principal**:
  1. Cliente envia `POST /webhooks` com `customerId`, `url` (protocolo HTTPS) e a lista de `events` (status de interesse).
  2. O sistema valida os campos via Zod schema e gera uma *secret* aleatória única para o endpoint (`[09:21] Sofia`, `[09:23] Sofia`).
  3. Salva a subscrição na tabela `webhook_subscriptions` e retorna o cadastro completo contendo a *secret* gerada (`[09:31] Marcos`).
- **Fluxos alternativos e exceções**:
  - Se a URL informada não utilizar o esquema `https://`, a requisição é rejeitada com erro `WEBHOOK_INVALID_URL` (HTTP 400) (`[09:23] Sofia`).
- **Erros previstos**: `WEBHOOK_INVALID_URL` (400), `WEBHOOK_SECRET_REQUIRED` (400).
- **Prioridade**: Alta

---

#### [FR-002] Enfileiramento Atômico do Evento na Mudança de Status (Outbox Pattern)
- **Descrição**: Enfileira o evento de notificação de forma atômica no banco de dados durante a transição de status do pedido (`[09:06] Diego`).
- **Fluxo principal**:
  1. Quando `OrderService.changeStatus` é invocado em `src/modules/orders/order.service.ts`, o sistema abre uma transação SQL `this.prisma.$transaction` (`[09:04] Bruno`, `[09:06] Diego`).
  2. Atualiza a ordem, histórico e estoque.
  3. Verifica se existem webhooks ativos cadastrados para o `customerId` interessados no novo status (`[09:33] Bruno`).
  4. Renderiza o snapshot estático JSON do payload e insere uma linha na tabela `webhook_outbox` com `status = 'PENDING'` e `eventId = UUIDv4` (`[09:06] Diego`, `[09:25] Diego`, `[09:52] Larissa`).
- **Fluxos alternativos e exceções**:
  - Se nenhum webhook ativo estiver cadastrado para aquele status, o evento não é inserido economizando espaço (`[09:34] Bruno`).
  - Se a transação SQL principal sofrer rollback, a gravação na outbox é cancelada junto (`[09:06] Diego`).
- **Erros previstos**: Erros de banco de dados SQL.
- **Prioridade**: Alta

---

#### [FR-003] Processamento Assíncrono e Polling pelo Worker
- **Descrição**: Processo separado que lê periodicamente a outbox e dispara as chamadas HTTP (`[09:08] Diego`, `[09:11] Diego`).
- **Fluxo principal**:
  1. O processo `src/worker.ts` roda continuamente executando polling na tabela `webhook_outbox` a cada 2 segundos (`[09:08] Diego`, `[09:09] Diego`).
  2. Consulta eventos pendentes por ordem de criação (`created_at ASC`).
  3. Dispara a requisição HTTP `POST` para a URL cadastrada com os headers de assinatura e identificação (`[09:20] Sofia`, `[09:44] Diego`).
  4. Se o cliente responder HTTP 2xx em até 10 segundos, marca o evento na outbox como `DELIVERED` e salva o log (`[09:08] Diego`, `[09:42] Diego`).
- **Fluxos alternativos e exceções**:
  - Se o cliente responder HTTP != 2xx ou exceder 10s de timeout, o evento é marcado como falha e encaminhado para retry (`[09:42] Diego`).
- **Erros previstos**: `WEBHOOK_DELIVERY_FAILED` (502).
- **Prioridade**: Alta

---

#### [FR-004] Assinatura HMAC-SHA256 e Validação de Integridade
- **Descrição**: Assina o payload JSON de cada webhook enviado garantindo a autenticidade e integridade da mensagem (`[09:20] Sofia`).
- **Fluxo principal**:
  1. O worker calcula o hash HMAC usando a chave secreta única do endpoint e o algoritmo SHA-256 sobre o corpo JSON da mensagem (`[09:20] Sofia`).
  2. Envia a assinatura no header `X-Signature` da requisição HTTP (`[09:20] Sofia`).
- **Fluxos alternativos e exceções**:
  - Se o snapshot do payload exceder 64KB, a tentativa é abortada com erro `WEBHOOK_MAX_PAYLOAD_EXCEEDED` (`[09:23] Sofia`, `[09:24] Diego`).
- **Erros previstos**: `WEBHOOK_MAX_PAYLOAD_EXCEEDED` (422).
- **Prioridade**: Alta

---

#### [FR-005] Rotação de Chaves Secretas com Grace Period de 24 Horas
- **Descrição**: Permite a renovação da chave secreta de um webhook mantendo a chave antiga ativa temporariamente por 24 horas (`[09:21] Sofia`).
- **Fluxo principal**:
  1. Cliente solicita `POST /webhooks/:id/rotate-secret`.
  2. O sistema gera uma nova *secret*, passa a antiga para `previousSecret` e define `previousSecretExpiresAt` para 24 horas no futuro (`[09:21] Sofia`).
  3. Durante 24h, assinaturas geradas com qualquer uma das duas chaves são aceitas (`[09:21] Sofia`).
  4. Após 24 horas, a chave anterior é definitivamente invalidada (`[09:21] Sofia`).
- **Erros previstos**: `WEBHOOK_NOT_FOUND` (404).
- **Prioridade**: Média

---

#### [FR-006] Resiliência e Política de Retry com Backoff Exponencial
- **Descrição**: Retenta o envio de notificações com falha em intervalos crescentes ao longo de ~15 horas (`[09:15] Diego`, `[09:17] Diego`).
- **Fluxo principal**:
  1. Em caso de falha HTTP ou timeout (>10s), o worker incrementa `attempts` (`[09:42] Diego`).
  2. Se `attempts < 5`, recalcula `next_retry_at` aplicando a escala: 1 min, 5 min, 30 min, 2 horas, 12 horas (`[09:17] Diego`).
  3. Evento permanece na outbox com status `FAILED` aguardando a data/hora da próxima tentativa (`[09:08] Diego`).
- **Erros previstos**: `WEBHOOK_DELIVERY_FAILED` (502).
- **Prioridade**: Alta

---

#### [FR-007] Transferência para Dead Letter Queue (DLQ) e Replay Admin
- **Descrição**: Isola mensagens com falhas permanentes e fornece endpoint administrativo para reprocessamento manual (`[09:18] Diego`).
- **Fluxo principal**:
  1. Ao falhar na 5ª tentativa, o evento é movido da `webhook_outbox` para a tabela `webhook_dead_letter` (`[09:18] Diego`).
  2. Usuário com role `ADMIN` dispara `POST /admin/webhooks/dead-letter/:id/replay` (`[09:18] Diego`, `[09:36] Sofia`).
  3. Middleware `requireRole('ADMIN')` valida o token e o evento é recolocado na outbox com status `PENDING` (`[09:18] Diego`, `[09:36] Sofia`).
  4. Ação grava log de auditoria no Pino logger (`[09:36] Sofia`).
- **Erros previstos**: `WEBHOOK_UNAUTHORIZED_REPLAY` (403), `WEBHOOK_NOT_FOUND` (404).
- **Prioridade**: Alta

---

#### [FR-008] Consulta de Histórico de Entregas pelo Cliente
- **Descrição**: Permite que o cliente B2B consulte o histórico de tentativas e status dos webhooks disparados (`[09:34] Marcos`).
- **Fluxo principal**:
  1. Cliente envia `GET /webhooks/:id/deliveries`.
  2. Sistema retorna lista paginada com status HTTP, tempo de resposta em ms, timestamp e id do evento (`[09:34] Marcos`).
- **Erros previstos**: `WEBHOOK_NOT_FOUND` (404).
- **Prioridade**: Média

---

### Requisitos não funcionais

**Performance**
- Latência fim a fim inferior a 10 segundos no pior caso (`[09:02] Marcos`, `[09:10] Larissa`).
- Timeout máximo de chamada HTTP do worker limitado a 10 segundos por tentativa (`[09:42] Diego`).

**Disponibilidade**
- Garantir a execução contínua do worker em processo isolado da API principal (`[09:11] Diego`).

**Segurança e autorização**
- Assinatura obrigatória HMAC-SHA256 no header `X-Signature` (`[09:20] Sofia`).
- Protocolo HTTPS (`https://`) obrigatório na URL do webhook validado por schema Zod (`[09:23] Sofia`).
- Endpoint de replay da DLQ restrito a usuários com a role `ADMIN` com audit log (`[09:36] Sofia`).

**Observabilidade**
- Logs estruturados em formato JSON utilizando a instância centralizada do Pino (`src/shared/logger/index.ts`) com `event_id`, `webhook_id`, `duration_ms` e `status_code` (`[09:29] Bruno`).

**Confiabilidade e integridade de dados**
- Escrita atômica na tabela outbox garantida por transação Prisma em `OrderService.changeStatus` (`[09:06] Diego`).
- Header `X-Event-Id` enviado com UUID v4 para deduplicação e idempotência no cliente B2B (`[09:25] Diego`).

**Compatibilidade e portabilidade**
- Compatível com a stack existente Node.js >= 20, Express 4.21, Prisma 5.22 e MySQL.

---

### Arquitetura e abordagem

**Abordagem**
- Padrão Outbox no MySQL com worker em processo separado rodando polling a cada 2 segundos (`[09:06] Diego`, `[09:08] Diego`).

**Componentes**
- **API Backend**: Express 4.21 gerenciando rotas CRUD de webhooks e transações de pedidos (`src/server.ts`).
- **Worker Process**: Processo Node.js isolado (`src/worker.ts`) executando polling e disparos HTTP.
- **Banco de Dados MySQL**: Armazena as tabelas `orders`, `webhook_subscriptions`, `webhook_outbox`, `webhook_delivery_logs` e `webhook_dead_letter` via Prisma ORM.

**Integrações**
- Notificações outbound HTTP `POST` para os endpoints cadastrados pelos clientes B2B (*Atlas Comercial*, *MaxDistribuição*, *Nova Cargo*).

---

### Decisões e trade-offs

#### Decisão: Padrão Outbox no MySQL
- **Justificativa**: Garante consistência atômica entre o pedido e a notificação sem subir nova infraestrutura (`[09:06] Diego`, `[09:07] Diego`).
- **Trade-off**: Aumenta I/O de escrita no MySQL e exige processo de polling (`[09:07] Bruno`).

#### Decisão: Worker em Processo Separado
- **Justificativa**: Evita que reinicializações da API HTTP parem a fila de envios de webhooks (`[09:11] Diego`).
- **Trade-off**: Exige gerenciar a execução de dois processos Node.js independentes (`src/server.ts` e `src/worker.ts`) (`[09:11] Larissa`).

#### Decisão: Rotação de Secrets com Grace Period de 24 Horas
- **Justificativa**: Evita rejeição de mensagens nos clientes B2B durante a troca de chaves (`[09:21] Sofia`).
- **Trade-off**: Mantém a chave secreta antiga válida por 24 horas em paralelo (`[09:21] Sofia`).

---

### Dependências

#### Dependência Técnica: Método Transacional no Prisma ORM
- Inserção atômica exige a disponibilidade e funcionamento do método `this.prisma.$transaction` em `src/modules/orders/order.service.ts`.

#### Dependência Organizacional: Revisão de Segurança por Sofia
- Reserva obrigatória de 2 dias úteis na Sprint 3 para revisão do código de segurança de HMAC e rotação pela engenheira Sofia antes do deploy (`[09:46] Larissa`, `[09:46] Sofia`).

---

### Riscos e mitigação

#### [Risco 1] Perda do cliente B2B Atlas Comercial por atraso na entrega da funcionalidade
- **Probabilidade**: Alta
- **Impacto**: Alto
- **Mitigação**:
  - Alocar o desenvolvimento em 3 Sprints com escopo fechado (`[09:46] Larissa`).
  - Cortar itens opcionais do MVP (como alertas por e-mail e dashboard visual) (`[09:37] Larissa`, `[09:40] Larissa`).
- **Plano de contingência**: Manter foco estrito no escopo do MVP de 3 Sprints com janela final de revisão de segurança (`[09:46] Larissa`, `[09:46] Sofia`).

#### [Risco 2] Trava do Worker por Endpoints de Clientes Inacessíveis ou Lentos
- **Probabilidade**: Alta
- **Impacto**: Alto
- **Mitigação**:
  - Configuração de timeout rígido de 10 segundos por requisição HTTP (`[09:42] Diego`).
  - Isolamento de mensagens falhas após 5 tentativas em Dead Letter Queue separada (`[09:18] Diego`).
- **Plano de contingência**: Isolamento automático do evento na Dead Letter Queue (DLQ) após a 5ª tentativa sem sucesso (`[09:18] Diego`).

---

### Critérios de aceitação

- ☐ Latência total de notificação entregue em menos de 10 segundos.
- ☐ Eventos gravados na outbox exclusivamente dentro da mesma transação SQL de `OrderService.changeStatus`.
- ☐ Mínimo de 8 Requisitos Funcionais (FR-001 a FR-008) totalmente implementados e testados.
- ☐ Assinatura HMAC-SHA256 enviada no header `X-Signature` com secret única por endpoint.
- ☐ Suporte a rotação de secret com validade da chave antiga por 24 horas.
- ☐ URLs `http://` recusadas por validação Zod com o erro `WEBHOOK_INVALID_URL`.
- ☐ Replay manual da DLQ funcional e protegido pela role `ADMIN`.

---

### Testes e validação

**Tipos de teste obrigatórios**
- **Testes Unitários**: Testes para os algoritmos de geração de HMAC-SHA256, schemas de validação Zod e máquina de estados de pedidos.
- **Testes de Integração**: Testes automatizados via Vitest e Supertest (`tests/`) validando a transação SQL em `changeStatus`, a gravação na outbox e o comportamento do worker em `src/worker.ts`.
- **Testes de Segurança e Resiliência**: Validação de URLs inválidas, simulação de timeout HTTP (>10s) e testes de permissão no replay da DLQ.

**Estratégia de validação**
- Validação contínua através da suíte `npm test`, seguida por testes ponta a ponta em ambiente local/dev com servidores HTTP mock e a janela final de 2 dias de homologação de segurança conduzida por Sofia (`[09:46] Sofia`).
