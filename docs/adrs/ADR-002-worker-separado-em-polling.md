# ADR-002: Worker em Processo Separado em Polling a Cada 2 Segundos

- **Status**: Aceito (Accepted)
- **Data**: 2026-07-22
- **Participantes**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Contexto

Com a decisão de utilizar o padrão Outbox no MySQL (`ADR-001`), os eventos de mudança de status dos pedidos são gravados de forma assíncrona na tabela `webhook_outbox` (`[09:06] Diego`). É necessário definir o mecanismo pelo qual esses eventos pendentes serão lidos da outbox e disparados via requisições HTTP para os endpoints dos clientes B2B.

O principal requisito de negócio é que as notificações sejam entregues em tempo "quase real", com latência máxima aceitável abaixo de 10 segundos (`[09:00] Marcos`, `[09:02] Marcos`). Além disso, a execução desse processamento de mensagens não pode concorrer diretamente por recursos do mesmo loop de eventos ou processo Node.js da API principal HTTP (`src/server.ts`), evitando que reinicializações da API interrompam os envios (`[09:11] Diego`).

---

## Decisão

Executar o processamento de webhooks em um **processo Node.js separado da API principal**, realizando **polling a cada 2 segundos** na tabela `webhook_outbox` (`[09:09] Diego`, `[09:11] Diego`).

O processo do worker terá como ponto de entrada o arquivo `src/worker.ts` (executado via script `npm run worker` no `package.json`), reaproveitando o mesmo banco de dados MySQL e schemas do Prisma, porém instanciando um cliente isolado do `PrismaClient` por se tratar de um processo Node.js independente (`[09:11] Larissa`, `[09:11] Bruno`, `[09:30] Bruno`).

A cada ciclo de 2 segundos, o worker consultará os registros da `webhook_outbox` com status `PENDING` ordenados por `created_at`, efetuará a chamada HTTP com timeout máximo de 10 segundos e atualizará o status do evento (`[09:08] Diego`, `[09:09] Diego`, `[09:42] Diego`).

---

## Alternativas Consideradas

### 1. Processo do Worker Rodando Embutido na Mesma Instância da API HTTP (`[09:11] Diego`)
- **Prós**: Não exige criar um novo script de execução ou gerenciar outro processo no ambiente de deploy.
- **Contras**: Se a instância da API principal reiniciar, o ciclo de envio de webhooks é interrompido (`[09:11] Diego`).
- **Motivo do Descarte**: Descartado por fragilidade operacional; se a API cair ou reiniciar, perde-se a execução do worker de envio (`[09:11] Diego`).

### 2. Uso de Triggers Nativas do MySQL para Notificação Reativa (`[09:09] Bruno`, `[09:09] Diego`)
- **Prós**: Notificação imediata ao banco sem necessidade de temporizador/loop de polling.
- **Contras**: O MySQL não possui sistema nativo de escuta/notificação pub-sub para processos externos tipo `NOTIFY/LISTEN` do PostgreSQL (`[09:09] Diego`). Triggers no MySQL executam apenas instruções SQL internas e não notificam aplicações Node.js ativamente (`[09:09] Diego`).
- **Motivo do Descarte**: Descartado por ser inviável no MySQL sem recorrer a mecanismos instáveis de arquivo ou rede externa no banco (`[09:09] Diego`).

### 3. Polling em Intervalos Menores (ex: 500ms ou 1s) ou Maiores (ex: 30s) (`[09:09] Diego`, `[09:10] Larissa`)
- **Prós**: Intervalos menores reduziriam ligeiramente a latência; intervalos maiores reduziriam a quantidade de consultas `SELECT` no MySQL.
- **Contras**: Intervalos muito curtos aumentam o *database overhead* sem ganho perceptível para o cliente; intervalos longos violam o SLA de 10 segundos do cliente *Atlas Comercial* (`[09:00] Marcos`, `[09:02] Marcos`).
- **Motivo do Descarte**: Polling de 2 segundos foi definido como o equilíbrio ideal, garantindo latência máxima de ~2 segundos no pior caso (`[09:09] Diego`, `[09:10] Larissa`).

---

## Consequências

### Positivas
- **Isolamento de Processos**: Reinicializações ou indisponibilidades no processo da API HTTP (`src/server.ts`) não afetam o envio dos webhooks (`[09:11] Diego`).
- **Garantia do SLA de Latência**: O polling de 2 segundos assegura que as notificações sejam enviadas muito abaixo do teto de 10 segundos exigido pelos clientes B2B (`[09:09] Diego`, `[09:10] Marcos`).
- **Resiliência com Timeout**: O limite de 10 segundos por requisição HTTP evita que conexões presas em clientes lentos travem a fila do worker (`[09:42] Diego`).
- **Simplicidade de Execução**: Criado um novo entry point `src/worker.ts` acionado facilmente por `npm run worker` (`[09:11] Larissa`).

### Negativas e Trade-offs
- **Consultas Constantes no MySQL**: O polling a cada 2 segundos realiza consultas periódicas na tabela `webhook_outbox`, mesmo quando não há eventos pendentes (`[09:08] Diego`).
- **Limitação de Ordenação Global**: Enquanto houver um único worker (*single-worker*), a ordenação por `created_at` e `order_id` é garantida; caso haja escala horizontal para múltiplos workers no futuro, será necessário implementar particionamento por `order_id` (`[09:12] Diego`, `[09:13] Diego`).
