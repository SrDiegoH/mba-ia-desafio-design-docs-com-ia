# ADR-006: Reuso máximo dos padrões e componentes já existentes no OMS

## Status

Aceito

## Contexto

O OMS já possui convenções de arquitetura consolidadas: estrutura modular por domínio, hierarquia de classes de erro, middleware de erro centralizado, middleware de autenticação/autorização, logger estruturado e padrão de schemas de validação. A nova feature de webhooks poderia introduzir convenções próprias (novo formato de erro, novo logger, nova estrutura de pastas), mas isso fragmentaria a base de código e aumentaria o custo de manutenção para um time pequeno.

Também é uma restrição de negócio implícita: a mudança de status de pedido (`OrderService.changeStatus`, `src/modules/orders/order.service.ts`, linhas 126-179) é uma transação crítica hoje síncrona — qualquer nova integração não pode comprometer sua confiabilidade ou exigir reescrevê-la do zero.

## Decisão

O módulo de webhooks (`src/modules/webhooks/`) reutiliza integralmente os padrões arquiteturais já estabelecidos no projeto, em vez de introduzir convenções próprias:

- **Estrutura de módulo** (controller, service, repository, routes, schemas), seguindo o padrão de `src/modules/orders/`, `src/modules/users/` etc. — [09:27, Bruno]: "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual."
- **Classes de erro** (`AppError` e subclasses de `src/shared/errors/`), com um novo prefixo de código dedicado (`WEBHOOK_*`) seguindo o mesmo espírito de `InsufficientStockError`/`InvalidStatusTransitionError` — [09:28-09:29, Bruno/Larissa].
- **Logger** (Pino), **middleware de erro centralizado** e **`requireRole`**, todos reaproveitados sem alteração — [09:29 e 09:36, Bruno/Larissa].
- **Padrão de identificadores** (UUID) e **validação** (schemas Zod), seguindo as mesmas convenções do restante do projeto — [09:23, 09:30, 09:51, Sofia/Larissa].

> "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro." — [09:30, Larissa]

O detalhamento de como cada um desses componentes se integra ao código existente — caminhos de arquivo exatos, nomes de classes de erro específicas, entry-point do worker — está em `docs/FDD.md`, seção "Integração com o Sistema Existente". Este ADR registra a decisão e o porquê; não repete esse detalhe.

## Alternativas Consideradas

**Criar convenções próprias para o módulo de webhooks** (novo padrão de erro, novo formato de log, estrutura de pastas diferente). Poderia, em tese, ser otimizado especificamente para as necessidades de um sistema orientado a eventos (diferente do padrão request/response síncrono dos demais módulos). Foi implicitamente descartada durante toda a reunião em favor da consistência com o restante da base — nenhum participante propôs alternativas de padrão próprio; a decisão de reuso foi tratada como natural e não controversa, e fechada explicitamente por Larissa — [09:27-09:30].

## Consequências

**Positivas**
- Time não precisa aprender/manter um segundo conjunto de convenções — qualquer desenvolvedor familiarizado com `orders` ou `users` já entende a estrutura de `webhooks`.
- Menor superfície de mudança no código existente: o middleware de erro, o logger e o middleware de autenticação não precisam de nenhuma alteração para suportar a nova feature.
- Erros do módulo de webhooks aparecem no mesmo formato de resposta já documentado/consumido pelos clientes de API (`{error:{code,message,details?}}`), reduzindo a curva de integração.

**Negativas**
- Padrões pensados originalmente para módulos CRUD síncronos (request/response) podem não se encaixar perfeitamente em todas as necessidades de um sistema assíncrono orientado a eventos (ex.: o padrão atual de controller/service não tem um conceito nativo de "worker" ou "processor" — foi necessário estender a convenção com um novo tipo de artefato, `webhook.worker.ts`).
- Os erros de negócio específicos já existentes no projeto (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`, `INACTIVE_PRODUCT`) já são nomeados por domínio, sem prefixo de módulo — só os erros-base genéricos (`NOT_FOUND`, `CONFLICT`) usam código genérico por classe HTTP. Adotar o prefixo `WEBHOOK_*` para todos os erros de negócio do módulo (não só os que hoje teriam nome próprio) é uma convenção ligeiramente mais explícita que a usada nos demais módulos — pequena inconsistência que vale documentar para não confundir times futuros.
