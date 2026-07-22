# ADR-005: Garantia de Entrega At-Least-Once e Idempotência via Header X-Event-Id

- **Status**: Aceito (Accepted)
- **Data**: 2026-07-22
- **Participantes**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Contexto

Em arquiteturas assíncronas baseadas em mensagens e retentativas de rede, o envio de notificações HTTP para endpoints externos pode sofrer instabilidades em que a resposta de confirmação (HTTP 2xx) do cliente B2B é perdida no trânsito, fazendo com que o worker assuma que o envio falhou e tente novamente (`[09:15] Diego`, `[09:24] Diego`).

Nesses cenários, é preciso definir o nível de garantia de entrega oferecido pela plataforma (*At-Least-Once* vs *Exactly-Once*) e fornecer aos clientes B2B os mecanismos necessários para identificarem e descartarem notificações duplicadas sem processar o mesmo evento duas vezes em seus sistemas internos (`[09:24] Diego`, `[09:25] Bruno`).

---

## Decisão

Garantir a semântica de entrega **At-Least-Once** (pelo menos uma vez) associada à emissão do header **`X-Event-Id` contendo um UUID v4 único por evento** para permitir idempotência e deduplicação no lado do cliente B2B (`[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`).

1. **Geração do UUID no Commit**: No momento em que a alteração de status ocorre no método `changeStatus` de `src/modules/orders/order.service.ts`, a gravação na tabela `webhook_outbox` gera um `eventId` único em formato UUID v4 (utilizando a biblioteca `uuid` declarada em `package.json` e o campo `@db.Char(36)` no `prisma/schema.prisma`) (`[09:25] Diego`, `[09:51] Larissa`).
2. **Envio nos Headers do Webhook**: Toda requisição disparada pelo worker em `src/worker.ts` enviará o identificador imutável no header `X-Event-Id`, acompanhado de `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` (`[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia`).
3. **Idempotência no Cliente**: O cliente B2B assume a responsabilidade de registrar o `X-Event-Id` recebido e ignorar envios subsequentes que apresentem o mesmo ID, garantindo idempotência em seus domínios (`[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`).

---

## Alternativas Consideradas

### 1. Garantia de Entrega Exactly-Once (Exatamente Uma Vez) (`[09:25] Diego`)
- **Prós**: O cliente B2B nunca receberia um evento duplicado, simplificando a lógica do lado dele.
- **Contras**: Exige coordenação síncrona de transações distribuídas bi-direcionais (ex: Two-Phase Commit ou protocolo de confirmação em duas etapas) entre o OMS e os servidores de cada cliente B2B, aumentando enormemente a complexidade e a latência (`[09:25] Diego`).
- **Motivo do Descarte**: Descartado por ser inviável e excessivamente complexo em notificações HTTP para sistemas externos desacoplados (`[09:25] Diego`).

### 2. Idempotência Baseada Apenas no ID do Pedido (`order_id`) ou Status (`[09:25] Bruno`, `[09:25] Diego`)
- **Prós**: Evita criar um header e UUID adicional para o evento.
- **Contras**: Se um pedido mudar de status para `SHIPPED`, for cancelado e posteriormente recriado/atualizado, o cliente B2B não conseguiria diferenciar se o recebimento é uma retentativa de rede do evento anterior ou uma nova transição legíitima (`[09:25] Diego`).
- **Motivo do Descarte**: Descartado por causar ambiguidade no controle de estado dos clientes B2B (`[09:25] Diego`).

---

## Consequências

### Positivas
- **Padrão de Mercado**: Adota o padrão amplamente utilizado pela indústria em APIs de webhooks (Stripe, GitHub), facilitando a integração pelos times de engenharia dos clientes B2B (`[09:25] Diego`, `[09:26] Marcos`).
- **Desacoplamento e Robustez**: Permite que o worker realize retentativas sem receio de corromper o estado dos clientes B2B, desde que estes dedupiquem pelo `X-Event-Id` (`[09:24] Diego`, `[09:25] Diego`).
- **Rastreabilidade Ponta a Ponta**: O `X-Event-Id` serve como chave primária de correlação em logs de auditoria no worker e nos registros de entregas (`[09:25] Diego`).

### Negativas e Trade-offs
- **Responsabilidade Transferida ao Cliente**: Exige que os clientes B2B implementem uma tabela/cache de deduplicação pelo `X-Event-Id` do lado deles (`[09:25] Sofia`, `[09:25] Diego`).
- **Risco de Processamento Duplicado no Cliente**: Caso o cliente B2B não implemente o controle de idempotência, ele poderá processar a mesma notificação mais de uma vez em caso de retentativas de rede (`[09:24] Diego`).
