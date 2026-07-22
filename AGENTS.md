# AGENTS.md — Guia de Contexto do Projeto para Agentes de IA

> [!IMPORTANT]
> **Propósito deste Arquivo**: Este documento serve como fonte da verdade e contexto completo do projeto para qualquer agente de Inteligência Artificial (ex: Antigravity / Gemini / Claude) atuando neste repositório. Ele reúne o escopo do desafio, a análise detalhada da transcrição técnica, o mapeamento da codebase existente e as regras operacionais obrigatórias.

---

## 1. Contexto Geral & Desafio do MBA

- **Origem do Projeto**: Este projeto é um desafio prático de MBA ("Da Reunião ao Documento: Design Docs Gerados por IA").
- **Documento Base de Referência**: O enunciado original do desafio e todas as suas diretrizes/critérios de aceite encontram-se preservados no arquivo [`ENUNCIADO.md`](file:///Users/leandromeira/Dev/mba-ia-desafio-design-docs-com-ia/ENUNCIADO.md). O arquivo `README.md` foi reservado para a documentação do processo final do aluno.
- **Objetivo Principal**: Utilizar a transcrição de uma reunião técnica real (`TRANSCRICAO.md`) e o código-fonte existente da aplicação Order Management System (OMS) para produzir um pacote completo e acionável de **Design Docs** (PRD, RFC, FDD, ADRs, Tracker e README do processo), seguindo integralmente as instruções contidas no [`ENUNCIADO.md`](file:///Users/leandromeira/Dev/mba-ia-desafio-design-docs-com-ia/ENUNCIADO.md).
- **Papel da IA**: A IA atua como ferramenta principal de produção ("executora sob mentoria/supervisão"), auxiliando na leitura da codebase, análise da transcrição, estruturação técnica e geração do conteúdo da documentação.
- **Restrição Absoluta (Código Intocável)**: A entrega deste projeto é **PURAMENTE DOCUMENTAL**. Os agentes de IA **NUNCA** devem alterar, criar ou remover arquivos de código fonte ou testes (`src/`, `prisma/`, `tests/`, `tsconfig.json`, `package.json`, etc.). O código existente serve estritamente como leitura de contexto e referência de integração.
- **Diretriz de Rastreabilidade e Validação Contínua (Zero Alucinação / Zero Invenção)**:
  - **Validação Pré e Pós-Geração**: **ANTES** de gerar qualquer documento de documentação e **APÓS** a geração, o agente de IA deve obrigatoriamente realizar um cross-check minucioso das informações com o arquivo [`TRANSCRICAO.md`](file:///Users/leandromeira/Dev/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md) e com o código fonte (`src/`, `prisma/schema.prisma`).
  - **Citação Obrigatória das Falas dos Participantes**: Em todos os arquivos de documentação produzidos (`PRD.md`, `RFC.md`, `FDD.md`, `ADR-*.md` e `TRACKER.md`), o agente deve **SEMPRE** referenciar explicitamente a fala da pessoa responsável pela decisão ou requisito, incluindo o timestamp e nome do falante (ex: `[09:17] Diego`, `[09:20] Sofia`, `[09:00] Marcos`, `[09:31] Larissa`, `[09:28] Bruno`). **É expressamente proibido inventar ou presumir requisitos sem citação comprovada.**

---

## 2. Análise Detalhada da Reunião Técnica (`TRANSCRICAO.md`)

### 2.1. Metadados & Participantes
- **Data/Horário**: Quinta-feira, 09:00 (Duração: ~55 minutos, via Google Meet).
- **Participantes**:
  1. **Larissa**: Tech Lead (conduziu a reunião e facilitou as decisões).
  2. **Marcos**: Product Manager (trouxe a dor dos clientes B2B e requisitos de negócio).
  3. **Bruno**: Engenheiro Pleno do time de Pedidos (foco na integração com o módulo de orders e transações).
  4. **Diego**: Engenheiro Sênior do time de Plataforma (propositor da arquitetura de Outbox, polling, worker e retry).
  5. **Sofia**: Engenheira de Segurança (definiu autenticação HMAC-SHA256, rotação de secrets, HTTPS e segurança).

### 2.2. Motivação de Negócio & Problema
- **Clientes Solicitantes**: Tres clientes B2B principais: *Atlas Comercial*, *MaxDistribuição* e *Nova Cargo*.
- **Problema Atual**: Os clientes fazem *polling* constante no endpoint `GET /orders` para verificar mudanças de status dos pedidos. Isso torna a integração lenta, custosa e ineficiente para ambas as partes.
- **Urgência**: A *Atlas Comercial* sinalizou risco de migração para um concorrente caso a solução de notificação em tempo real não seja entregue até o fim do trimestre.
- **Requisito de Latência Aceitável**: Notificações entregues em tempo "quase real", definido como **latência abaixo de 10 segundos**.
- **Estimativa de Entrega**: **3 Sprints** de desenvolvimento, incluindo janela obrigatória de 2 dias úteis no final para revisão de segurança pela Sofia.

### 2.3. Decisões Arquiteturais Chave Alinhadas
1. **Direção do Fluxo**: Webhooks exclusivamente **Outbound** (do OMS para os clientes B2B).
2. **Padrão Outbox no MySQL**:
   - Quando o status de um pedido muda em `OrderService.changeStatus`, a escrita na tabela `webhook_outbox` ocorre de forma atômica **dentro da mesma transação SQL** que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`.
   - *Alternativas descartadas*: Disparo HTTP síncrono no service (travaria a API e falharia transações) e Redis Streams / RabbitMQ (subir novo cluster geraria overengineering para um time pequeno).
3. **Worker Separado em Polling (2 segundos)**:
   - Processo Node.js isolado (`src/worker.ts`, executado por `npm run worker`), separado do processo principal da API (`src/server.ts`).
   - O worker executa polling a cada 2 segundos na tabela `webhook_outbox`, buscando eventos pendentes por ordem de criação (`created_at`).
   - Timeout máximo de cada requisição HTTP enviada ao cliente: **10 segundos** (se exceder, considera falha e incrementa retry).
   - Abre uma instância própria de `PrismaClient`.
   - *Alternativa descartada*: Triggers do MySQL (o MySQL não possui `NOTIFY/LISTEN` nativo para notificar processos externos).
4. **Resiliência, Retry e Dead Letter Queue (DLQ)**:
   - Política de **5 tentativas** de retry.
   - Progressão de backoff exponencial: **1 min, 5 min, 30 min, 2 horas, 12 horas** (cobrindo janela total de ~15 horas).
   - Se todas as 5 tentativas falharem, o evento é movido para a tabela `webhook_dead_letter` (DLQ).
   - Replay manual de eventos da DLQ feito via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`.
5. **Segurança & Assinatura de Payload**:
   - Assinatura no header `X-Signature` via **HMAC-SHA256** sobre o corpo do request JSON.
   - Chave secreta (*secret*) **única por endpoint** de webhook cadastrado pelo cliente.
   - Suporte a rotação de secret com **grace period de 24 horas** (a secret antiga permanece válida por 24h após a emissão de uma nova).
   - Validação obrigatória de protocolo **HTTPS** (`https://`) na URL do webhook via Zod.
   - Limite máximo de tamanho do payload do evento: **64 KB** (se exceder, gera erro e cancela envio).
6. **Semântica de Entrega & Idempotência**:
   - Garantia de entrega **At-Least-Once**.
   - Header `X-Event-Id` enviado com UUID único por evento (gerado na inserção da outbox) para permitir que o cliente realize deduplicação do seu lado.
   - Demais headers obrigatórios enviados no request do webhook: `X-Event-Id`, `X-Signature`, `X-Timestamp` (timestamp do envio), `X-Webhook-Id` (ID da configuração do webhook), `Content-Type: application/json`.
7. **Snapshot de Payload**:
   - O payload do evento é renderizado e armazenado como JSON estático (snapshot) no momento em que a transação de mudança de status é commitada.
   - Payload enxuto em formato JSON: contem `event_id`, `event_type` (`order.status_changed`), `timestamp` (formato ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` (sem a lista de itens, para manter o payload leve).
8. **Identificadores Únicos**:
   - Todas as tabelas de webhook utilizam **UUID (v4)** como chave primária, seguindo a convenção de todo o banco de dados do projeto.
9. **Controle de Acesso & Autorização**:
   - Endpoints CRUD de configuração de webhook (`/webhooks`) utilizam autenticação JWT padrão (roles `OPERATOR` ou `ADMIN`).
   - Endpoint de replay da DLQ (`/admin/webhooks/dead-letter/:id/replay`) exige estritamente a role `ADMIN`, reaproveitando o middleware `requireRole('ADMIN')` e gravando log de auditoria.

### 2.4. Itens Explicitamente Fora de Escopo / Adiados
- **Fora de Escopo (Descartado)**: Envio de e-mail de notificação ao cliente quando o webhook apresentar falhas consecutivas (postergado para fases futuras).
- **Fora de Escopo (Descartado)**: Dashboard visual / Painel de interface gráfica para visualização de webhooks (escopo do time de Frontend/produto separado; neste projeto entregamos apenas os endpoints REST HTTP).
- **Fora de Escopo (Descartado)**: Arquivamento/limpeza automática de registros entregues na outbox após 30 dias (identificado, porém fora da release inicial).
- **Fora de Escopo (Descartado)**: Disparo de webhooks em chamadas síncronas HTTP ou adoção de mensageria dedicada (Redis/RabbitMQ).
- **Ponto em Aberto / Adiado**: *Rate limiting* de envio por cliente no worker (decidido observar a volumetria pós-deploy antes de implementar limitação).
- **Limitação Conhecida**: Garantia de ordenação global (*ordering*). Enquanto houver um único worker (*single-worker*), a ordenação por `order_id` e `created_at` é mantida; caso ocorra escalabilidade horizontal no futuro, particionamento por `order_id` será necessário.

---

## 3. Scan Técnico da Codebase Existente

### 3.1. Arquitetura & Stack Tecnológica
- **Runtime**: Node.js (>= 20.0.0) com suporte a módulos ESM.
- **Linguagem**: TypeScript 5.6 com tipagem estrita (`tsconfig.json`).
- **Framework Web**: Express 4.21.
- **ORM / Banco de Dados**: Prisma ORM 5.22 interagindo com MySQL.
- **Validação de Schemas**: Zod 3.23.
- **Logging**: Pino 9.5 & Pino-HTTP 10.3 (centralizado em `src/shared/logger/index.ts`).
- **Testes**: Vitest 2.1 & Supertest 7.0.
- **Criptografia & Utils**: `bcrypt` (5.1), `jsonwebtoken` (9.0), `uuid` (11.0).

### 3.2. Estrutura de Diretórios do Projeto
```
.
├── .env.example                         # Variáveis de ambiente de exemplo
├── README.md                            # Documentação do desafio / a ser substituída pelo processo
├── TRANSCRICAO.md                       # Transcrição literal da reunião técnica (Fonte Primária)
├── docker-compose.yml                   # Container MySQL de desenvolvimento
├── package.json                         # Dependências e scripts npm
├── prisma/
│   ├── schema.prisma                    # Modelos do banco de dados (User, Customer, Product, Order, etc.)
│   └── seed.ts                          # Seed inicial de dados para dev/testes
├── src/
│   ├── app.ts                           # Instanciação do Express e registro de middlewares/rotas
│   ├── server.ts                        # Entry point do servidor HTTP (API principal)
│   ├── config/                          # Configurações de ambiente (env.ts)
│   ├── middlewares/                     # Middlewares compartilhados
│   │   ├── auth.middleware.ts           # Middleware authenticate e requireRole
│   │   ├── error.middleware.ts          # Error handler centralizado (AppError, Zod, Prisma)
│   │   ├── request-logger.middleware.ts # Logging HTTP com Pino
│   │   └── validate.middleware.ts       # Validação de schemas Zod
│   ├── modules/                         # Módulos de domínio
│   │   ├── auth/                        # Autenticação e emissão de JWT
│   │   ├── customers/                   # Cadastro e busca de clientes
│   │   ├── orders/                      # Gestão de pedidos, itens e transições de status
│   │   │   ├── order.controller.ts
│   │   │   ├── order.repository.ts
│   │   │   ├── order.routes.ts
│   │   │   ├── order.schemas.ts
│   │   │   ├── order.service.ts         # Contém changeStatus com transação Prisma
│   │   │   └── order.status.ts          # Máquina de estados (canTransition, stock checks)
│   │   ├── products/                    # Cadastro e controle de estoque de produtos
│   │   └── users/                       # Gestão de usuários (ADMIN / OPERATOR)
│   ├── routes/                          # Roteador central (index.ts)
│   └── shared/                          # Recursos e utilitários globais
│       ├── errors/                      # AppError e subclasses (NotFoundError, ConflictError, etc.)
│       ├── http/                        # Helpers de resposta HTTP e paginação
│       └── logger/                      # Instância singleton do Pino Logger
└── tests/                               # Testes de integração com Vitest
```

### 3.3. Entidades de Banco de Dados (`prisma/schema.prisma`)
- **`User`**: Usuários do sistema (`id`, `email`, `passwordHash`, `name`, `role` [`ADMIN`, `OPERATOR`]).
- **`Customer`**: Clientes do e-commerce (`id`, `name`, `email`, `phone`, `document`, `address` [JSON]).
- **`Product`**: Produtos (`id`, `sku`, `name`, `priceCents`, `stockQuantity`, `active`).
- **`Order`**: Pedidos (`id`, `orderNumber`, `customerId`, `status` [`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`], `subtotalCents`, `discountCents`, `totalCents`, `createdById`).
- **`OrderItem`**: Itens do pedido (`id`, `orderId`, `productId`, `quantity`, `unitPriceCents`, `totalCents`).
- **`OrderStatusHistory`**: Histórico de mudanças de status (`id`, `orderId`, `fromStatus`, `toStatus`, `changedById`, `reason`, `changedAt`).

### 3.4. Componentes do Código Relevantes para a Integração da Feature
1. **`src/modules/orders/order.service.ts` (`changeStatus`)**:
   - Método em `OrderService` executado dentro de `this.prisma.$transaction(async (tx) => { ... })`.
   - Atualiza o pedido, cria o histórico em `OrderStatusHistory` e credita/decreta estoque.
   - Ponto de integração obrigatório: chamar a função auxiliar de enfileiramento de webhooks (ex: `publishWebhookEvent(tx, order, from, to)`) injetando o cliente de transação `tx`.
2. **`src/shared/errors/app-error.ts` & `src/shared/errors/index.ts`**:
   - Base para erros da aplicação. Todas as exceções de webhook devem estender ou reutilizar o formato `AppError`, adotando o prefixo `WEBHOOK_` para os códigos de erro (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_MAX_PAYLOAD_EXCEEDED`).
3. **`src/middlewares/auth.middleware.ts`**:
   - Exporta os middlewares `authenticate` e `requireRole`.
   - O endpoint de replay de DLQ utilizará `requireRole('ADMIN')`.
4. **`src/middlewares/error.middleware.ts`**:
   - Intercepta `AppError`, erros do Zod (`ZodError`) e erros conhecidos do Prisma (`PrismaClientKnownRequestError`), retornando respostas HTTP padronizadas com código JSON e status adequado.
5. **`src/shared/logger/index.ts`**:
   - Instância centralizada do Pino Logger para registrar tentativas do worker, envios com sucesso/falha e logs de auditoria de replay.

---

## 4. Pacote de Entregáveis Exigidos (Design Docs)

O objetivo principal deste projeto é preencher e entregar o pacote completo de documentação na pasta `docs/` e na raiz:

| Documento | Caminho | Escopo / Papel | Exigências Principais |
| --- | --- | --- | --- |
| **PRD** | `docs/PRD.md` | Produto / Negócio (*Por que e o quê?*) | Requisitos funcionais (mín. 8), RNF, métricas quantitativas, fora de escopo (mín. 2 itens), análise de riscos (mín. 2 riscos com probabilidade, impacto e mitigação). |
| **RFC** | `docs/RFC.md` | Arquitetura (*Como pretendemos resolver?*) | Formato RFC curto (2 a 4 págs). Metadados com revisores, TL;DR, alternativas consideradas (mín. 2 descartadas com trade-offs), questões em aberto (mín. 2), links para ADRs. |
| **FDD** | `docs/FDD.md` | Implementação (*Como construir em detalhe?*) | Fluxos (outbox, worker, retry, DLQ), contratos de **mín. 4 endpoints HTTP** (com exemplos de request/response e status code), matriz de erros com prefixo `WEBHOOK_`, observabilidade, e a **seção obrigatória "Integração com o sistema existente"** citando ao menos 4 caminhos de arquivos reais da codebase (`src/...`). |
| **ADRs** | `docs/adrs/ADR-NNN-*.md` | Decisões isoladas (*Por que decidimos assim?*) | **Entre 5 e 8 arquivos** no formato MADR (`ADR-001-outbox-no-mysql.md`, etc.). Deve cobrir ao menos 5 das 6 decisões chave da reunião. Ao menos 1 ADR deve referenciar caminhos reais da codebase. |
| **Tracker** | `docs/TRACKER.md` | Rastreabilidade (*De onde veio cada item?*) | Tabela markdown com colunas `ID`, `Documento`, `Tipo`, `Conteúdo (resumo)`, `Fonte` (`TRANSCRICAO` ou `CODIGO`), `Localização` (`[hh:mm] Nome` ou caminho de arquivo). Cobertura de mín. 80% dos itens, mín. 70% linhas de Transcrição e mín. 5 linhas de Código. |
| **README** | `README.md` | Documentação do Processo | Substituir o enunciado pelas seções: Sobre o desafio, Ferramentas de IA utilizadas, Workflow adotado, Prompts customizados (mín. 2 em blocos de código), Iterações e ajustes (mín. 2 relatos concretos de correção), Como navegar a entrega. |

---

## 5. Mapeamento de Aulas & Materiais de Referência do Curso (`Design Docs com IA/`)

A pasta [`Design Docs com IA/`](file:///Users/leandromeira/Dev/mba-ia-desafio-design-docs-com-ia/Design%20Docs%20com%20IA) contém a estrutura de módulos, aulas e materiais do curso de MBA. Para a criação de cada tipo de documento exigido no desafio, devem ser consultadas as seguintes aulas e referências:

### 5.1. PRD (Product Requirement Document) — `docs/PRD.md`
- **Módulo do Curso**: `Design Docs com IA/03 - Documentação na era da IA/`
- **Aulas de Referência**:
  - `03 - PRDs - Product Requirement Document`
  - `04 - Seções de um PRD`
  - `05 - PRDs de Alto Nivel`
  - `06 - PRDs de Feature e casos de uso`
  - `07 - Principais Seções em um PRD de Feature`
  - `08 - Exemplo 1 de PRD`
  - `09 - Exemplo 2 de PRD e JSON Opcional`
  - `10 - Prompt para geração de PRD`
- **Material de Apoio**: `Design Docs com IA/01 - Material/01 - Templates PRD.md`

### 5.2. RFC (Request for Comments) — `docs/RFC.md` & HLD
- **Módulo do Curso**: `Design Docs com IA/04 - Design e Arquitetura/`
- **Aulas de Referência**:
  - `01 - Documentos de Design e Arquitetura`
  - `02 - Documentos comuns`
  - `03 - High Level Design`
  - `04 - Exemplo de um High Level Design Document`
  - `05 - Prompt para High Level Design`
  - `09 - Realizando Deep Research`
  - `10 - Adaptando Deep Research a um novo formato`
- **Material de Apoio**: `Design Docs com IA/01 - Material/02 - Design e Arquitetura.md`

### 5.3. FDD (Feature Design Document) — `docs/FDD.md`
- **Módulo do Curso**: `Design Docs com IA/04 - Design e Arquitetura/`
- **Aulas de Referência**:
  - `06 - Feature Design Doc`
  - `07 - Exemplo de um FDD`
  - `08 - Prompt para FDD`
  - `11 - Gerando um FDD a partir da Deep Research`
- **Suporte de Diagramas (`Design Docs com IA/05 - Diagramas - Design e Arquitetura/`)**:
  - `02 - Introdução aos diagramas C4` a `06 - C4-Code`
  - `08 - Gerando diagrama C4`
  - `09 - Diagramas Mermaid`
  - `10 - Flowchart e Sequence Diagram`
  - `12 - State Diagram`
  - `13 - Gerando diagramas Mermaid com IA`

### 5.4. ADRs (Architecture Decision Records) — `docs/adrs/ADR-NNN-*.md`
- **Módulo do Curso**: `Design Docs com IA/06 - Architecture Decision Record/`
- **Aulas de Referência**:
  - `01 - Introdução a ADRs`
  - `02 - Estrutura clássica`
  - `03 - MADRs` (Modelo de ADR adotado)
  - `04 - Status, Metadados e Fluxos`
  - `05 - Boas práticas e dicas importantes`
  - `06 - Quando e recomendável utilizar ADRs`
  - `09 - Agente de mapeamento na prática`
  - `10 - Criando potenciais ADRs`
  - `11 - Gerando ADRs com IA`
  - `12 - Linkando e relacionando ADRs`

### 5.5. Tracker de Rastreabilidade e README do Processo — `docs/TRACKER.md` & `README.md`
- **Módulos do Curso**: `Design Docs com IA/03 - Documentação na era da IA/` & `Design Docs com IA/07 - Engineering Guidelines/`
- **Aulas de Referência**:
  - `02 - Tipos de documentação e Design Docs`
  - `01 - Software Development Guidelines`
  - `02 - Guidelines na era da IA`
  - `03 - Seções sugeridas de um guideline`
  - `04 - Project Stack`
  - `06/07 - Comando para geracao de guidelines`
  - `09 - Classificação de Documentos e Considerações finais`
- **Materiais de Apoio**:
  - `Design Docs com IA/01 - Material/03 - Engineering Guidelines.md`
  - `Design Docs com IA/01 - Material/04 - Classificação de Documentos.md`
  - `Design Docs com IA/01 - Material/05 - Plugins Claude Code.md`

---

## 6. Regras Operacionais para Agentes de IA

1. **Nunca editar o código-fonte**: Sob nenhuma hipótese tente criar, modificar ou remover arquivos nas pastas `src/`, `prisma/` ou `tests/`. Toda a contribuição é exclusivamente restrita a arquivos de documentação Markdown (`docs/`, `README.md`).
2. **Consultar antes de escrever**: Ao gerar os documentos técnicos, consulte as definições exatas do banco (`prisma/schema.prisma`), os serviços existentes (`src/modules/orders/order.service.ts`), os tipos e os middlewares para garantir 100% de consistência de nomenclatura de variáveis, tipos e caminhos.
3. **Não inventar requisitos**: Adira estritamente às decisões fechadas na transcrição `TRANSCRICAO.md`. Se um ponto não foi decidido ou foi descartado na reunião, registre-o na seção apropriada ("Fora de escopo" ou "Questões em aberto"), em vez de inventar uma solução não combinada.
4. **Manter consistência entre documentos**:
   - O **PRD** trata de produto e negócio.
   - O **RFC** trata da proposta arquitetural e alternativas.
   - Os **ADRs** documentam decisões isoladas e definitivas.
   - O **FDD** detalha o contrato de API, payloads, diagramas de sequência e integração real no código.
   - O **Tracker** conecta todos os itens anteriores à sua origem factual.
5. **Confirmação Obrigatória Pré e Pós-Geração (`TRANSCRICAO.md`)**:
   - **Antes da Geração**: O agente DEVE obrigatoriamente ler os trechos correspondentes da `TRANSCRICAO.md` para mapear os fatos exatos.
   - **Após a Geração**: O agente DEVE revisar linha a linha o documento produzido para garantir que nenhum dado, parâmetro ou comportamento foi inventado pela IA.
6. **Citação Obrigatória das Falas das Pessoas**:
   - Em toda documentação gerada, o agente DEVE SEMPRE citar a fala da pessoa que deu a instrução ou tomou a decisão técnica, indicando o timestamp e o nome do participante (ex: `[09:17] Diego: fala...`, `[09:20] Sofia`, `[09:00] Marcos`, `[09:31] Larissa`, `[09:28] Bruno`).

