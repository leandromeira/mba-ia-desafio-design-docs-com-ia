# ADR-001: Adoção do Padrão Outbox no MySQL para Enfileiramento de Webhooks

- **Status**: Aceito (Accepted)
- **Data**: 2026-07-22
- **Participantes**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Contexto

O Order Management System (OMS) opera em produção com módulo de pedidos gerenciado via Prisma ORM e banco MySQL. A empresa precisa notificar clientes B2B (*Atlas Comercial*, *MaxDistribuição*, *Nova Cargo*) em tempo quase real (latência < 10 segundos) sempre que o status de um pedido for alterado (`[09:00] Marcos`, `[09:02] Marcos`).

Atualmente, o método `changeStatus` em `src/modules/orders/order.service.ts` executa a transição de status dentro de uma transação SQL (`this.prisma.$transaction`), atualizando a tabela `orders`, inserindo o histórico em `order_status_history` e debitando/creditando estoque em `products` (`[09:04] Bruno`).

O desafio é registrar e enfileirar o evento de notificação do webhook sem travar a requisição HTTP da API e garantindo que **nenhum evento seja gravado se a transação do pedido falhar (rollback)**, e que **nenhum pedido alterado fique sem seu evento correspondente (commit)** (`[09:06] Diego`).

---

## Decisão

Adotar o **Padrão Outbox no MySQL** (`[09:06] Diego`, `[09:08] Larissa`).

Sempre que o status de um pedido for alterado no método `changeStatus` de `src/modules/orders/order.service.ts`, a gravação do evento de notificação será realizada inserindo um novo registro na tabela `webhook_outbox` (definida em `prisma/schema.prisma`) **dentro da mesma transação SQL (`tx`)** da mudança de status (`[09:06] Diego`).

O payload da notificação é renderizado e persistido em JSON como um snapshot estático no momento em que a transação é commitada (`[09:52] Larissa`).

---

## Alternativas Consideradas

### 1. Disparo HTTP Síncrono no `OrderService` (`[09:03] Larissa`, `[09:04] Bruno`)
- **Prós**: Não exige tabela auxiliar de outbox nem worker de processamento.
- **Contras**: Clientes lentos ou fora do ar travavam a API principal e causavam rollback indevido de pedidos válidos (`[09:04] Bruno`). Viola o isolamento de domínios e adiciona alta latência no endpoint `PATCH /orders/:id/status` (`[09:04] Bruno`, `[09:06] Diego`).
- **Motivo do Descarte**: Descartado por colocar em risco a disponibilidade e estabilidade do OMS (`[09:04] Bruno`, `[09:06] Diego`).

### 2. Mensageria Dedicada Externa (Redis Streams / RabbitMQ) (`[09:07] Larissa`, `[09:07] Diego`)
- **Prós**: Alta performance e desacoplamento nativo por mensagens.
- **Contras**: Exige provisionar, monitorar e manter um novo cluster de infraestrutura em produção, gerando *overengineering* para um time pequeno (`[09:07] Diego`).
- **Motivo do Descarte**: Descartado por adicionar complexidade e custos de infraestrutura desnecessários quando o MySQL existente atende a demanda (`[09:07] Diego`).

### 3. Triggers Nativas da tabela `orders` no MySQL (`[09:09] Bruno`, `[09:09] Diego`)
- **Prós**: Lógica centralizada diretamente no motor do banco de dados.
- **Contras**: O MySQL não possui sistema nativo de publicação/notificação para processos externos (tipo `NOTIFY/LISTEN` do Postgres) (`[09:09] Diego`). Triggers no MySQL só executam SQL e não notificam workers de forma reativa (`[09:09] Diego`).
- **Motivo do Descarte**: Descartado por inviabilidade técnica de notificação externa via MySQL (`[09:09] Diego`).

---

## Consequências

### Positivas
- **Consistência Atômica Total**: A escrita na tabela `webhook_outbox` faz parte da transação SQL gerenciada pelo Prisma (`tx`). Se o pedido for commitado, o evento entra na outbox; se ocorrer rollback, o evento é desfeito (`[09:06] Diego`).
- **Desacoplamento e Performance**: A requisição HTTP da API de pedidos finaliza imediatamente sem aguardar o envio HTTP da notificação para o cliente B2B (`[09:08] Diego`).
- **Simplicidade de Infraestrutura**: Reusa 100% o banco de dados MySQL e o Prisma ORM já existentes na aplicação (`[09:07] Diego`).
- **Snapshot de Estado Preservado**: O payload da notificação reflete exatamente o estado do pedido no momento em que a alteração de status ocorreu (`[09:52] Larissa`).

### Negativas e Trade-offs
- **Carga Adicional de Escrita no MySQL**: Acrescenta uma operação de `INSERT` na tabela `webhook_outbox` dentro da transação de `changeStatus` (`[09:07] Bruno`).
- **Necessidade de Processo Worker**: Requer a criação e execução de um processo de worker separado (`src/worker.ts`) para realizar polling na outbox (`[09:08] Larissa`, `[09:11] Diego`).
