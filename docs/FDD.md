# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

O OMS não possui hoje nenhum mecanismo de notificação externa, eventos ou filas. A mudança de status de um pedido acontece inteiramente dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179), numa única transação Prisma (`this.prisma.$transaction`) que atualiza `orders`, ajusta `stockQuantity` de produtos e grava auditoria em `OrderStatusHistory`. Este documento detalha como estender esse fluxo para publicar eventos de webhook sem comprometer a atomicidade e a performance dessa transação, e como um worker assíncrono processa e entrega essas notificações. Decisões de arquitetura já fechadas estão registradas em `docs/adrs/`; este documento assume essas decisões como dadas e foca em como implementá-las.

## Objetivos Técnicos

- Registrar eventos de mudança de status de forma atômica com a transação de negócio (outbox pattern, [[../adrs/ADR-001-outbox-pattern-no-mysql]]).
- Entregar notificações HTTP autenticadas (HMAC-SHA256) dentro de latência-alvo < 10s desde a mudança de status.
- Garantir resiliência a indisponibilidade de terceiros via retry com backoff e DLQ, sem perda silenciosa de eventos.
- Expor CRUD de configuração de webhook e histórico de entregas para os clientes gerenciarem suas integrações.
- Não introduzir infraestrutura nova além do MySQL já existente; reaproveitar módulos, middlewares e convenções do projeto.

## Escopo e Exclusões

**Em escopo:** CRUD de configuração de webhook por cliente; outbox e worker de processamento; retry/backoff/DLQ; replay administrativo de DLQ; assinatura HMAC-SHA256 com rotação de secret; histórico de entregas (`deliveries`); filtro de eventos por status.

**Fora de escopo** (ver `docs/PRD.md` para detalhamento de negócio):
- Notificação por e-mail em caso de falha repetida de entrega — adiado [09:37-09:38, Marcos/Larissa].
- Dashboard visual de gestão de webhooks — adiado, fica a cargo de projeto futuro do time de frontend [09:39-09:40, Marcos/Larissa].
- Múltiplos workers em paralelo / particionamento de processamento — adiado [09:13, Diego].
- Rate limiting de envio — questão em aberto, não implementada nesta fase [09:38-09:39, Diego/Larissa].
- Arquivamento/purga de linhas entregues da outbox após 30 dias — fora do escopo desta feature [09:08, Diego].
- Endurecimento de permissões do CRUD de configuração de webhook (hoje qualquer usuário autenticado pode gerenciar) — adiado [09:37, Sofia].

## Fluxos Detalhados

### 1. Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` é chamado normalmente (validação de transição via `canTransition`, débito/reposição de estoque conforme hoje).
2. Antes do fim da transação, após `tx.order.update` e `tx.orderStatusHistory.create`, o service chama uma nova função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o mesmo client transacional (`tx`) já em uso.
3. `publishWebhookEvent`:
   a. Busca as configurações de webhook ativas do `customerId` do pedido cujo filtro de status inclua `toStatus`.
   b. Se não houver nenhuma configuração correspondente, não insere nada (early return) — nenhum overhead para clientes sem webhook cadastrado.
   c. Para cada configuração encontrada, monta o payload (snapshot, ver [[../adrs/ADR-007-snapshot-do-payload-no-momento-do-evento]]) e insere uma linha em `webhook_outbox` com `status = 'PENDING'`, `event_id` (UUID), `webhook_config_id`, payload serializado, `created_at`.
4. Se qualquer etapa falhar, a exceção propaga e a transação inteira (incluindo a mudança de status) sofre rollback — consistente com o restante do método `changeStatus`, que já usa esse padrão para erros de estoque/transição inválida.
5. Transação commita: mudança de status e evento(s) de webhook estão persistidos atomicamente.

### 2. Processamento pelo worker

1. `src/worker.ts` inicia, instancia seu próprio `PrismaClient` (mesmo `DATABASE_URL` da API) e entra em loop de polling a cada 2s.
2. A cada ciclo, busca um lote de linhas em `webhook_outbox` com `status = 'PENDING'` (ou `RETRYING` com `next_attempt_at <= now()`), ordenadas por `created_at`, com um `LIMIT` pequeno (ex. 20).
3. Marca as linhas selecionadas como `status = 'PROCESSING'` (evita reprocessamento concorrente caso o worker seja escalado no futuro).
4. Para cada evento:
   a. Monta o request HTTP: `POST <url_do_webhook>`, headers (ver Contratos Públicos), body = payload snapshot.
   b. Calcula `X-Signature` (HMAC-SHA256 do body, usando a secret ativa da configuração de webhook).
   c. Envia com timeout de 10s.
   d. Se resposta HTTP `2xx`: marca evento como `status = 'DELIVERED'`, grava linha em histórico de deliveries (`webhook_delivery`) com status code, tempo de resposta, corpo da resposta (truncado se necessário).
   e. Se falha (timeout, erro de rede, status `4xx`/`5xx`): incrementa `attempt_count`; se `attempt_count < 5`, agenda `next_attempt_at` conforme a tabela de backoff e marca `status = 'RETRYING'`; se `attempt_count >= 5`, move o evento para `webhook_dead_letter` (payload, motivo da última falha, timestamp) e remove/marca como `FAILED` na outbox.
5. Loop continua indefinidamente até o processo ser encerrado (graceful shutdown análogo ao de `src/server.ts`, tratando SIGINT/SIGTERM e aguardando o ciclo atual terminar antes de `prisma.$disconnect()`).

### 3. Retry com backoff

| Tentativa | Atraso desde a tentativa anterior |
| --- | --- |
| 1 (inicial) | imediata (primeiro ciclo de polling após inserção) |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 30 minutos |
| 5 | 2 horas |
| — (última reavaliação antes de DLQ) | 12 horas |

Fonte: [09:17, Diego] — decisão registrada em [[../adrs/ADR-003-politica-de-retry-com-backoff-e-dlq]]. Após a 5ª tentativa falhar, o evento é movido para DLQ (não há 6ª tentativa automática).

### 4. Dead Letter Queue e replay

1. Evento esgotado é inserido em `webhook_dead_letter` com `payload`, `failure_reason`, `failed_at`, `original_event_id`.
2. Administrador consulta a DLQ (endpoint de listagem, ver Contratos Públicos) e decide reprocessar.
3. `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: valida role `ADMIN` via `requireRole('ADMIN')`, recria uma linha em `webhook_outbox` com `status = 'PENDING'` e `attempt_count = 0`, usando o payload original preservado (snapshot).
4. Log de auditoria: `logger.info({adminId, deadLetterId, eventId}, 'webhook_dead_letter_replayed')` — reaproveita o Pino já configurado.

## Contratos Públicos

Todos os endpoints seguem o prefixo `/api/v1/webhooks` (ou `/api/v1/admin/webhooks` para o endpoint administrativo), autenticação via `authenticate` (JWT) e validação via schemas Zod, seguindo o padrão dos demais módulos.

### `POST /api/v1/webhooks`

Cria uma configuração de webhook para um cliente.

Request:
```json
{
  "customerId": "3f1a9c2e-...-uuid",
  "url": "https://cliente.exemplo.com/hooks/oms",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "b7e2f0a1-...-uuid",
  "customerId": "3f1a9c2e-...-uuid",
  "url": "https://cliente.exemplo.com/hooks/oms",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_9f8a7b6c5d4e3f2a1b0c",
  "active": true,
  "createdAt": "2026-07-19T14:03:00.000Z"
}
```

A `secret` é retornada **apenas na criação** (não é recuperável posteriormente via `GET`). URL não-`https` é rejeitada com `400 WEBHOOK_INVALID_URL`.

### `PATCH /api/v1/webhooks/:id`

Atualiza `url`, `statuses` ou `active` de uma configuração existente.

Request:
```json
{
  "statuses": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true
}
```

Response `200 OK`: mesmo formato do `POST` (sem o campo `secret`).

### `DELETE /api/v1/webhooks/:id`

Remove uma configuração de webhook. Response `204 No Content`. Eventos já na outbox/DLQ referentes a essa configuração não são removidos retroativamente (histórico preservado).

### `GET /api/v1/webhooks?customerId=:customerId`

Lista as configurações de webhook de um cliente.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "b7e2f0a1-...-uuid",
      "customerId": "3f1a9c2e-...-uuid",
      "url": "https://cliente.exemplo.com/hooks/oms",
      "statuses": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-19T14:03:00.000Z"
    }
  ],
  "meta": { "page": 1, "pageSize": 20, "total": 1 }
}
```
Paginação segue o padrão de `paginated()` já usado em `OrderService.list` (`src/shared/http/response.ts`).

### `POST /api/v1/webhooks/:id/rotate-secret`

Gera uma nova secret para o endpoint, mantendo a anterior válida por 24h (grace period).

Response `200 OK`:
```json
{
  "id": "b7e2f0a1-...-uuid",
  "secret": "whsec_1a2b3c4d5e6f7a8b9c0d",
  "previousSecretExpiresAt": "2026-07-20T14:03:00.000Z"
}
```

### `GET /api/v1/webhooks/:id/deliveries`

Histórico das últimas 100 tentativas de entrega da configuração.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "d4c3b2a1-...-uuid",
      "eventId": "e1f2a3b4-...-uuid",
      "eventType": "order.status_changed",
      "status": "DELIVERED",
      "httpStatusCode": 200,
      "responseTimeMs": 184,
      "attemptNumber": 1,
      "sentAt": "2026-07-19T14:05:02.000Z"
    },
    {
      "id": "c3b2a1d4-...-uuid",
      "eventId": "f2a3b4c5-...-uuid",
      "eventType": "order.status_changed",
      "status": "FAILED",
      "httpStatusCode": 503,
      "responseTimeMs": 10000,
      "attemptNumber": 5,
      "sentAt": "2026-07-19T02:00:00.000Z"
    }
  ]
}
```

### `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Reenfileira um evento da DLQ. Exige `requireRole('ADMIN')`.

Response `200 OK`:
```json
{
  "outboxEventId": "a1b2c3d4-...-uuid",
  "status": "PENDING",
  "replayedAt": "2026-07-19T15:00:00.000Z",
  "replayedBy": "9d8c7b6a-...-uuid"
}
```

Response `403 Forbidden` (usuário sem role `ADMIN`):
```json
{ "error": { "code": "FORBIDDEN", "message": "Insufficient permissions" } }
```

### Payload de evento enviado ao cliente (`POST <url_do_webhook>`)

Headers:
```
Content-Type: application/json
X-Event-Id: e1f2a3b4-...-uuid
X-Webhook-Id: b7e2f0a1-...-uuid
X-Signature: sha256=7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e
X-Timestamp: 2026-07-19T14:05:00.000Z
```

Body:
```json
{
  "event_id": "e1f2a3b4-...-uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-19T14:05:00.000Z",
  "order_id": "1a2b3c4d-...-uuid",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f1a9c2e-...-uuid",
  "total_cents": 15990
}
```

Fonte do formato: [09:43-09:45, Diego/Sofia]. Payload intencionalmente enxuto (sem `items`) — cliente que precisar de detalhes completos consulta `GET /orders/:id`.

## Matriz de Erros

| Código | Status HTTP | Cenário |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Configuração de webhook não encontrada (`GET`/`PATCH`/`DELETE`/`rotate-secret`/`deliveries` com `id` inexistente) |
| `WEBHOOK_INVALID_URL` | 400 | URL não é `https` ou não passa na validação de schema Zod — [09:23, Sofia] |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | Lista de `statuses` contém valor fora do enum `OrderStatus` |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Tentativa de processar entrega sem secret ativa configurada |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB — sistema falha explicitamente, não trunca — [09:23-09:24, Sofia] |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (erro interno do worker, não resposta HTTP a cliente externo) | Timeout de 10s excedido na chamada ao endpoint do cliente — [09:42, Diego] |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de `id` inexistente na `webhook_dead_letter` |
| `WEBHOOK_DUPLICATE_CONFIG` | 409 | Já existe configuração ativa para a mesma combinação `customerId` + `url` |

Todos os erros seguem o formato de resposta já padronizado pelo `error.middleware.ts`: `{ "error": { "code": string, "message": string, "details"?: any } }`.

## Estratégias de Resiliência

- **Timeout de entrega**: 10 segundos por tentativa HTTP — [09:42, Diego].
- **Retry com backoff exponencial**: 5 tentativas (1m/5m/30m/2h/12h) — [[../adrs/ADR-003-politica-de-retry-com-backoff-e-dlq]].
- **Dead Letter Queue**: eventos esgotados não são perdidos, ficam disponíveis para replay manual auditado.
- **Isolamento de processo**: worker roda separado da API — falha de um não derruba o outro ([[../adrs/ADR-002-worker-em-processo-separado-com-polling]]).
- **Rollback conjunto**: inserção do evento na outbox é parte da mesma transação de `changeStatus` — falha na outbox reverte também a mudança de status, e vice-versa.
- **Falha explícita em payload grande**: em vez de fallback silencioso (truncar), o sistema recusa e sinaliza `WEBHOOK_PAYLOAD_TOO_LARGE` — [[../adrs/ADR-008-tls-obrigatorio-e-limite-de-payload]].
- **Idempotência do lado do cliente**: `X-Event-Id` permite ao cliente descartar entregas duplicadas (garantia at-least-once, não exactly-once) — [[../adrs/ADR-005-garantia-at-least-once-com-x-event-id]].

## Observabilidade

- **Logs**: reuso do Pino (`src/shared/logger/index.ts`), seguindo a convenção de eventos nomeados em snake_case já usada no projeto (`http_request`, `server_started`). Novos eventos: `webhook_event_published` (na inserção da outbox), `webhook_delivery_attempted`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_moved_to_dead_letter`, `webhook_dead_letter_replayed`. Todos os logs incluem `eventId` e `webhookConfigId` para correlação.
- **Métricas** (a expor via contadores/histogramas, mecanismo de coleta a definir em implementação): contagem de eventos publicados por status de destino; taxa de sucesso/falha de entrega por cliente; tempo de resposta (`responseTimeMs`) dos endpoints de clientes; profundidade da fila de eventos `PENDING`/`RETRYING` na outbox (proxy direto de saúde do worker); contagem de eventos movidos para DLQ.
- **Tracing/correlação**: `X-Event-Id` (gerado na outbox) e `X-Request-Id` (já existente via `request-logger.middleware.ts` para requests da API) permitem rastrear uma notificação da origem (mudança de status) até a entrega (ou falha) nos logs do worker.

## Dependências e Compatibilidade

- MySQL (já em uso via Prisma) — nenhuma infraestrutura nova.
- Cliente HTTP para chamadas de saída do worker (a ser adicionado como dependência, ex. `undici`/`axios`/`node-fetch` — não há cliente HTTP de saída no projeto hoje; avaliar na implementação, sem impacto nas decisões de arquitetura deste documento).
- Novas variáveis de ambiente a adicionar em `src/config/env.ts` (mesmo padrão Zod já usado), ex.: `WEBHOOK_MAX_RETRIES`, `WEBHOOK_TIMEOUT_MS`, `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_MAX_PAYLOAD_BYTES`.
- Migração Prisma adicionando os models `WebhookConfig`, `WebhookOutboxEvent`, `WebhookDeadLetter`, `WebhookDelivery` (nomes definitivos a confirmar na implementação; nenhum desses models existe hoje em `prisma/schema.prisma`).
- Compatível com o fluxo de graceful shutdown já usado em `src/server.ts` (SIGINT/SIGTERM); `src/worker.ts` deve implementar o mesmo padrão.

## Integração com o Sistema Existente

Esta seção nomeia os pontos exatos de integração com o código já existente no repositório.

1. **`src/modules/orders/order.service.ts` (método `changeStatus`, linhas 126-179)** — ponto de integração central. A transação `this.prisma.$transaction` já atualiza `orders`, ajusta estoque via `debitStock`/`replenishStock` e cria o registro em `tx.orderStatusHistory.create`. A nova função `publishWebhookEvent(tx, order, fromStatus, toStatus)` é chamada dentro dessa mesma transação, reaproveitando o client `tx` já existente, imediatamente após a criação do histórico de status. Nenhuma mudança na assinatura pública do método `changeStatus` é necessária — a chamada é interna ao service.

2. **`src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`** — as novas classes de erro do módulo de webhooks seguem o mesmo padrão de `InvalidStatusTransitionError extends ConflictError` e `InsufficientStockError extends UnprocessableEntityError`: por exemplo, `WebhookNotFoundError extends NotFoundError`, `WebhookPayloadTooLargeError extends UnprocessableEntityError`, cada uma associada a um código da matriz de erros (`WEBHOOK_*`). Todas herdam de `AppError`, então nenhuma alteração é necessária nessas classes-base.

3. **`src/middlewares/error.middleware.ts`** — já trata qualquer subclasse de `AppError` (incluindo as novas classes de erro de webhook) formatando a resposta no padrão `{error:{code,message,details?}}`, já trata `ZodError` (validação dos schemas de webhook) e erros do Prisma (`P2002`/`P2025`, relevantes para conflitos de configuração duplicada e configurações não encontradas). Nenhuma modificação é necessária neste arquivo.

4. **`src/middlewares/auth.middleware.ts` (`authenticate` e `requireRole`)** — os endpoints de CRUD de configuração de webhook usam apenas `authenticate` (qualquer usuário autenticado, decisão vigente para esta fase — [09:36-09:37, Marcos/Sofia]); o endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` usa `authenticate` + `requireRole('ADMIN')`, seguindo exatamente o padrão já usado em rotas administrativas de outros módulos (ex. `user.routes.ts`).

5. **`src/shared/logger/index.ts`** — o worker e os controllers do módulo de webhooks importam o `logger` Pino singleton já configurado (com `redact.paths` já cobrindo campos sensíveis) e seguem a convenção de eventos nomeados em snake_case usada no restante do projeto, sem qualquer alteração de configuração do logger.

6. **`prisma/schema.prisma`** — recebe os novos models do módulo de webhooks (`WebhookConfig`, `WebhookOutboxEvent`, `WebhookDeadLetter`, `WebhookDelivery`), com `id` em UUID seguindo o padrão de todos os models existentes (`Order`, `Customer`, `Product`, etc.), e relação (`customerId`) referenciando o model `Customer` já existente.

7. **`src/app.ts` e `src/routes/index.ts`** — o novo `buildWebhookRouter(controller)` é registrado em `src/routes/index.ts` sob o prefixo `/webhooks` (e `/admin/webhooks` para o endpoint de replay), seguindo o mesmo padrão de composição usado para `/orders`, `/customers`, `/products`; a instanciação do controller/service/repository de webhooks é adicionada a `buildControllers` em `src/app.ts`, no mesmo padrão dos demais módulos.

8. **`src/config/env.ts`** — novas variáveis de ambiente do worker (timeout, intervalo de polling, limites) são adicionadas ao `envSchema` (Zod) existente, seguindo o mesmo padrão de validação obrigatória no boot (`process.exit(1)` se inválido).

## Critérios de Aceite Técnicos

- Uma mudança de status que corresponda a um filtro de webhook ativo resulta em uma linha em `webhook_outbox` na mesma transação do `changeStatus`, verificável por teste de integração que force rollback e confirme ausência do evento.
- O worker entrega um evento pendente em até um ciclo de polling (2s) após sua inserção, para o caso de endpoint saudável.
- Uma falha de entrega é reagendada conforme a tabela de backoff, e o evento migra para `webhook_dead_letter` exatamente após a 5ª tentativa falhar.
- Toda entrega inclui `X-Signature` válida (verificável recalculando o HMAC com a secret correspondente) e `X-Event-Id` estável entre tentativas do mesmo evento.
- Todos os endpoints HTTP do módulo respondem no formato de erro padrão do projeto para os cenários da matriz de erros.
- Endpoint de replay de DLQ retorna `403` para usuários sem role `ADMIN` e registra log de auditoria em replays bem-sucedidos.

## Riscos e Mitigação

- **Risco**: aumento de latência/complexidade na transação de `changeStatus` por causa do `INSERT` adicional na outbox. **Mitigação**: operação é um `INSERT` local à mesma transação MySQL, sem chamada de rede; medir impacto em teste de carga antes do rollout.
- **Risco**: worker parado (crash) atrasa todas as notificações pendentes além da meta de 10s. **Mitigação**: processo dedicado com restart automático (fora do escopo de código, mas requisito operacional); métrica de profundidade de fila permite alerta.
- **Risco**: cliente não implementa deduplicação e trata entregas duplicadas como eventos de negócio distintos. **Mitigação**: documentação do contrato deixa claro o uso de `X-Event-Id`; fora do controle direto do OMS.
