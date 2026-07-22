# ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)

- **Status**: Aceito (Accepted)
- **Data**: 2026-07-22
- **Participantes**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Contexto

Ao enviar notificações de webhook outbound para endpoints de clientes B2B, a aplicação pode se deparar com falhas temporárias de rede, *timeouts* (superiores a 10 segundos) ou indisponibilidades prolongadas nos servidores dos clientes (`[09:15] Diego`, `[09:16] Diego`). 

É necessário estabelecer um mecanismo de resiliência que tente reentregar o evento em intervalos espaçados para absorver instabilidades passageiras, sem que retentativas infinitas fiquem presas indefinidamente na tabela `webhook_outbox` sobrecarregando o worker (`[09:15] Diego`). Além disso, eventos que esgotarem as tentativas precisam ser isolados para investigação e reprocessamento controlado por administradores (`[09:18] Diego`, `[09:36] Sofia`).

---

## Decisão

Adotar uma política de **5 tentativas de retry com backoff exponencial**, seguida de transferência para uma **Dead Letter Queue (DLQ)** em tabela separada `webhook_dead_letter` (`[09:15] Diego`, `[09:17] Larissa`, `[09:18] Diego`).

1. **Progressão do Backoff Exponencial**: As 5 tentativas seguirão os intervalos de **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** após cada falha (`[09:17] Diego`). A janela total de retentativas cobre aproximadamente 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`, `[09:17] Marcos`).
2. **Transferência para DLQ**: Caso a 5ª tentativa falhe, o evento é movido da `webhook_outbox` para a tabela `webhook_dead_letter` (modelada no `prisma/schema.prisma`), gravando o payload completo, o motivo detalhado da falha e o timestamp do esgotamento (`[09:18] Diego`).
3. **Replay Manual Restrito a ADMIN**: O reprocessamento de eventos da DLQ será feito exclusivamente via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento na outbox como `PENDING` (`[09:18] Diego`, `[09:35] Larissa`). Este endpoint exige obrigatoriamente a *role* `ADMIN` reaproveitando o middleware `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` e grava log de auditoria via Pino logger (`src/shared/logger/index.ts`) (`[09:36] Sofia`, `[09:36] Larissa`).

---

## Alternativas Consideradas

### 1. Retry Indefinito com Backoff Limitado (`[09:15] Diego`)
- **Prós**: Nunca abandona o evento, tentando a entrega perpetuamente até o cliente voltar.
- **Contras**: Se o cliente B2B desativar o servidor permanentemente ou mudar de URL sem avisar, o evento ficará acumulado na outbox tentando para sempre, inflando o banco de dados e desperdiçando processamento do worker (`[09:15] Diego`).
- **Motivo do Descarte**: Descartado por gerar degradação contínua na fila de outbox e consumo ineficiente de recursos (`[09:15] Diego`).

### 2. Política Agressiva de Apenas 3 Tentativas (`[09:15] Bruno`, `[09:16] Bruno`)
- **Prós**: Libera mais rápido os recursos e declara falha em curto espaço de tempo (~30 minutos).
- **Contras**: Clientes em manutenção planejada de poucas horas teriam suas notificações canceladas e movidas para DLQ prematuramente (`[09:16] Diego`).
- **Motivo do Descarte**: Descartado por ser rígido demais; clientes B2B possuem janelas de manutenção de 2 horas ou mais (`[09:16] Diego`).

### 3. Marcação de Falha Permanente na Própria Tabela Outbox (`[09:17] Larissa`, `[09:18] Diego`)
- **Prós**: Evita criar uma nova tabela no banco MySQL.
- **Contras**: Mistura eventos pendentes com eventos mortos na mesma tabela, tornando as consultas de polling mais lentas e dificultando a auditoria e debug (`[09:18] Diego`).
- **Motivo do Descarte**: Descartado em prol de uma tabela dedicada `webhook_dead_letter`, que mantém a leitura da outbox limpa e otimizada por índices (`[09:18] Diego`).

---

## Consequências

### Positivas
- **Alta Taxa de Sucesso em Instabilidades**: Cobertura de 15 horas de tentativas previne perda de notificações em manutenções e quedas temporárias de rede dos clientes B2B (`[09:17] Diego`, `[09:17] Marcos`).
- **Isolamento de Eventos Problemáticos**: Eventos permanentemente falhos são movidos para `webhook_dead_letter`, mantendo a tabela principal `webhook_outbox` leve e performática (`[09:18] Diego`).
- **Segurança e Auditoria**: Replay manual controlado via `POST /admin/webhooks/dead-letter/:id/replay`, protegido por `requireRole('ADMIN')` e auditado no logger Pino de `src/shared/logger/index.ts` (`[09:36] Sofia`).

### Negativas e Trade-offs
- **Complexidade Operacional de Replay**: Exige intervenção manual da equipe de suporte/administração via API REST para reprocessar eventos retidos na DLQ (`[09:18] Diego`, `[09:36] Sofia`).
- **Acúmulo de Dados na DLQ**: Tabela `webhook_dead_letter` exige monitoramento para não acumular registros antigos indefinidamente (`[09:18] Diego`).
