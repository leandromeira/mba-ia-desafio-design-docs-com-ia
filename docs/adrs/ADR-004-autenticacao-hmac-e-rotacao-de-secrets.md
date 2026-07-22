# ADR-004: Autenticação via HMAC-SHA256, Secret por Endpoint e Rotação com Grace Period

- **Status**: Aceito (Accepted)
- **Data**: 2026-07-22
- **Participantes**: Larissa, Diego, Bruno, Sofia, Marcos

---

## Contexto

Ao enviar notificações de webhooks contendo dados de pedidos para serviços externos fora da infraestrutura da plataforma, é fundamental garantir que o receptor consiga autenticar a origem do evento e verificar a integridade da mensagem, comprovando que o payload não foi adulterado no trânsito (`[09:19] Sofia`).

Além disso, é necessário evitar vulnerabilidades de segurança como o uso de chaves compartilhadas globais — cuja exposição comprometeria todas as integrações da plataforma (`[09:21] Sofia`, `[09:22] Diego`) — e garantir uma transição suave sem indisponibilidade de serviço no momento em que os clientes B2B precisarem renovar suas chaves (`[09:21] Sofia`).

---

## Decisão

Adotar o padrão de assinatura **HMAC-SHA256** no header `X-Signature`, **chave secreta (*secret*) única por endpoint** de webhook, **suporte a rotação com *grace period* de 24 horas** e **validação obrigatória de protocolo HTTPS** (`[09:20] Sofia`, `[09:21] Sofia`, `[09:23] Sofia`).

1. **Assinatura HMAC-SHA256**: O worker renderiza o corpo da requisição JSON (limitado ao tamanho máximo de 64KB) e calcula um código HMAC usando o algoritmo SHA-256 e a *secret* do endpoint. A assinatura calculada é enviada no header `X-Signature` (`[09:20] Sofia`, `[09:24] Diego`).
2. **Secret Única por Cadastro**: Cada configuração de webhook na tabela `webhook_subscriptions` (em `prisma/schema.prisma`) possui uma *secret* própria gerada aleatoriamente no momento do cadastro (`[09:21] Sofia`).
3. **Rotação com Grace Period de 24 Horas**: Na solicitação de rotação de chave via API, a nova *secret* passa a assinar os envios e a *secret* antiga é mantida no campo `previousSecret` válida por exatamente 24 horas (`previousSecretExpiresAt`), permitindo que o cliente B2B atualize suas chaves sem perdas de notificações (`[09:21] Sofia`). Após 24 horas, a chave antiga perde a validade automaticamente (`[09:21] Sofia`).
4. **TLS/HTTPS Obrigatório**: A URL cadastrada para o webhook deve usar obrigatoriamente o esquema `https://`, sendo validada por regras do Zod seguindo o padrão existente em `src/middlewares/validate.middleware.ts`. Tentativas de cadastrar URLs `http://` são recusadas imediatamente com o erro `WEBHOOK_INVALID_URL` (`[09:23] Sofia`).

---

## Alternativas Consideradas

### 1. Secret Global da Plataforma para Todos os Clientes (`[09:21] Sofia`)
- **Prós**: Simplifica a gestão de chaves no banco de dados e no código do worker.
- **Contras**: Se um único cliente vazar a chave secreta em seus logs ou repositórios, a segurança de todas as notificações de todos os clientes B2B da plataforma é comprometida (`[09:21] Sofia`, `[09:22] Diego`).
- **Motivo do Descarte**: Descartado por violar os requisitos mínimos de isolamento e segurança de dados (`[09:21] Sofia`).

### 2. Autenticação via Token Estático no Header (Bearer Token) (`[09:19] Sofia`, `[09:20] Sofia`)
- **Prós**: Implementação mais simples do lado do cliente B2B, exigindo apenas comparar uma string no header.
- **Contras**: Não garante a integridade do payload da mensagem. Um atacante homem-no-meio (*man-in-the-middle*) poderia alterar os dados do pedido (ex: status ou valores) mantendo o token original intacto (`[09:19] Sofia`).
- **Motivo do Descarte**: Descartado em prol do HMAC-SHA256, que assina cryptographicamente o corpo completo da requisição JSON (`[09:20] Sofia`).

### 3. Rotação Instantânea de Secret sem Período de Transição (`[09:21] Sofia`)
- **Prós**: Cancela imediatamente o acesso da chave antiga.
- **Contras**: Causa falhas de autenticação e rejeição imediata de webhooks no lado do cliente B2B durante o tempo em que suas aplicações demoram para atualizar a variável de ambiente ou banco local (`[09:21] Sofia`).
- **Motivo do Descarte**: Descartado em prol do *grace period* de 24 horas, garantindo disponibilidade contínua durante a troca de chaves (`[09:21] Sofia`).

---

## Consequências

### Positivas
- **Autenticidade e Integridade Garantidas**: Os clientes B2B conseguem verificar com segurança que o webhook veio da plataforma e que o payload não sofreu alterações (`[09:19] Sofia`, `[09:20] Sofia`).
- **Isolamento de Segurança**: O vazamento acidental da *secret* de um cliente não afeta as demais integrações de outros clientes B2B (`[09:21] Sofia`).
- **Zero Indisponibilidade na Rotação**: A janela de transição de 24 horas garante que a troca de chaves ocorra sem perda de eventos (`[09:21] Sofia`).
- **Trânsito Seguro com HTTPS**: Impedido o tráfego de dados sensíveis de pedidos em texto puro através da validação Zod de TLS obrigatório (`[09:23] Sofia`).

### Negativas e Trade-offs
- **Complexidade de Validação para o Cliente**: Exige que o receptor implemente a lógica de verificação de assinatura HMAC-SHA256 (`[09:20] Sofia`).
- **Atribuição de Múltiplos Campos no Banco**: Exige armazenar `secret`, `previousSecret` e `previousSecretExpiresAt` na modelagem da tabela no Prisma (`[09:21] Sofia`).
