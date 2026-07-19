# ADR-006: Reuso máximo dos padrões e componentes já existentes no OMS

## Status

Aceito

## Contexto

O OMS já possui convenções de arquitetura consolidadas: estrutura modular por domínio, hierarquia de classes de erro, middleware de erro centralizado, middleware de autenticação/autorização, logger estruturado e padrão de schemas de validação. A nova feature de webhooks poderia introduzir convenções próprias (novo formato de erro, novo logger, nova estrutura de pastas), mas isso fragmentaria a base de código e aumentaria o custo de manutenção para um time pequeno.

Também é uma restrição de negócio implícita: a mudança de status de pedido (`OrderService.changeStatus`, `src/modules/orders/order.service.ts`, linhas 126-179) é uma transação crítica hoje síncrona — qualquer nova integração não pode comprometer sua confiabilidade ou exigir reescrevê-la do zero.

## Decisão

O módulo de webhooks (`src/modules/webhooks/`) reutiliza integralmente os padrões arquiteturais já estabelecidos no projeto:

- **Estrutura de módulo**: controller, service, repository, routes e schemas, seguindo exatamente o padrão de `src/modules/orders/`, `src/modules/users/` etc. — [09:27, Bruno]: "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual."
- **Entry-point do worker**: `src/worker.ts`, análogo a `src/server.ts`, com lógica de processamento em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) — [09:11 e 09:28, Larissa/Bruno].
- **Classes de erro**: reuso de `AppError` (`src/shared/errors/app-error.ts`) e do padrão de subclasses (`ConflictError`, `NotFoundError`, `UnprocessableEntityError`), seguindo o modelo de `InsufficientStockError`/`InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), com um novo prefixo de código dedicado: `WEBHOOK_*` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) — [09:28-09:29, Bruno/Larissa].
- **Logger**: reuso direto do Pino já configurado em `src/shared/logger/index.ts`, sem alterações — [09:29, Bruno]: "já tá no projeto inteiro. Não vamos botar nada novo."
- **Middleware de erro centralizado**: `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e erros do Prisma; nenhuma alteração é necessária para que erros do módulo de webhooks sejam formatados corretamente — [09:29, Bruno]: "já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada."
- **Autorização**: reuso do middleware `requireRole` (`src/middlewares/auth.middleware.ts`) para proteger o endpoint administrativo de replay de DLQ — [09:36, Larissa]: "a gente reaproveita o requireRole que já existe."
- **Padrão de identificadores**: UUID para todas as novas entidades (`webhook_outbox`, `webhook_dead_letter`, configuração de webhook), seguindo o padrão já usado em todo o schema Prisma — [09:51, Larissa]: "UUID, segue o padrão do resto do projeto. Tudo é uuid."
- **Validação**: schemas Zod, seguindo o padrão de `*.schemas.ts` de cada módulo existente — [09:23 e 09:30, Sofia/Larissa].

> "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro." — [09:30, Larissa]

O ponto de integração com o código existente é o método `changeStatus` do `OrderService`: a inserção do evento na `webhook_outbox` ocorre dentro da mesma transação já usada para atualizar `orders`, debitar/repor estoque e gravar `order_status_history` — [09:40-09:41, Bruno/Diego].

## Alternativas Consideradas

**Criar convenções próprias para o módulo de webhooks** (novo padrão de erro, novo formato de log, estrutura de pastas diferente). Poderia, em tese, ser otimizado especificamente para as necessidades de um sistema orientado a eventos (diferente do padrão request/response síncrono dos demais módulos). Foi implicitamente descartada durante toda a reunião em favor da consistência com o restante da base — nenhum participante propôs alternativas de padrão próprio; a decisão de reuso foi tratada como natural e não controversa, e fechada explicitamente por Larissa — [09:27-09:30].

## Consequências

**Positivas**
- Time não precisa aprender/manter um segundo conjunto de convenções — qualquer desenvolvedor familiarizado com `orders` ou `users` já entende a estrutura de `webhooks`.
- Menor superfície de mudança no código existente: o middleware de erro, o logger e o middleware de autenticação não precisam de nenhuma alteração para suportar a nova feature.
- Erros do módulo de webhooks aparecem no mesmo formato de resposta já documentado/consumido pelos clientes de API (`{error:{code,message,details?}}`), reduzindo a curva de integração.

**Negativas**
- Padrões pensados originalmente para módulos CRUD síncronos (request/response) podem não se encaixar perfeitamente em todas as necessidades de um sistema assíncrono orientado a eventos (ex.: o padrão atual de controller/service não tem um conceito nativo de "worker" ou "processor" — foi necessário estender a convenção com um novo tipo de artefato, `webhook.worker.ts`).
- Códigos de erro com prefixo `WEBHOOK_*` introduzem uma convenção de nomenclatura ligeiramente mais explícita que a usada em módulos anteriores (que majoritariamente usam códigos genéricos por classe HTTP, como `NOT_FOUND`/`CONFLICT`, sem prefixo de domínio) — pequena inconsistência que vale documentar para não confundir times futuros.
