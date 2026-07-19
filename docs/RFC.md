# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Aprovado (decisões fechadas em reunião técnica) |
| Data | Reunião técnica de definição, ~55min (ver `TRANSCRICAO.md`) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Eng. de Segurança) |
| Documentos relacionados | `docs/PRD.md`, `docs/FDD.md`, `docs/adrs/` |

## TL;DR

Vamos implementar um sistema de webhooks para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de pedido em tempo real (latência-alvo < 10s), substituindo o polling manual que fazem hoje contra `GET /orders`. A abordagem é baseada no padrão **Transactional Outbox** no próprio MySQL: a mudança de status em `OrderService.changeStatus` grava, na mesma transação, um evento em uma tabela `webhook_outbox`; um **worker em processo separado**, rodando em polling de 2s, lê esses eventos e entrega via HTTP com **assinatura HMAC-SHA256** por endpoint, **retry com backoff exponencial (5 tentativas) e DLQ**, e garantia de entrega **at-least-once** com deduplicação via `X-Event-Id`. A solução reaproveita ao máximo os padrões já existentes no projeto (estrutura de módulos, classes de erro, middleware de erro, `requireRole`, logger Pino), sem exigir nenhuma infraestrutura nova além do próprio MySQL.

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem hoje polling manual em `GET /orders` para saber quando o status de um pedido muda — não existe nenhum mecanismo de notificação externa no OMS [09:00, Marcos]. A Atlas sinalizou risco de churn caso a notificação em tempo real não seja entregue até o fim do trimestre [09:00, Marcos]. "Tempo real" foi definido pelo negócio como uma latência abaixo de 10 segundos entre a mudança de status e a chegada da notificação [09:02, Marcos].

A mudança de status hoje é feita de forma síncrona e transacional em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179): dentro de uma única transação Prisma, o método atualiza o status do pedido, ajusta estoque (débito ou reposição, conforme a transição) e grava um registro de auditoria em `OrderStatusHistory`. Não existe hoje nenhum mecanismo de eventos, fila ou publish assíncrono no sistema — este é exatamente o vácuo que a feature deve preencher.

O desafio de design central é: como notificar sistemas externos sobre uma mudança de estado interna sem (a) acoplar a disponibilidade de terceiros à confiabilidade da transação crítica de negócio, e (b) introduzir infraestrutura desproporcional ao tamanho do time e ao problema.

## Proposta Técnica

A solução tem três componentes principais:

**1. Registro de eventos via Transactional Outbox.** Uma nova tabela `webhook_outbox` recebe uma linha na mesma transação SQL que já atualiza `orders` e `order_status_history` dentro de `changeStatus`. Isso garante atomicidade sem coordenação distribuída: se a transação principal falha, o evento nunca existiu; se ela commita, o evento está garantidamente registrado. Detalhes em [[ADR-001-outbox-pattern-no-mysql]].

**2. Worker assíncrono em processo separado.** Um novo entry-point (`src/worker.ts`, análogo ao `src/server.ts` existente) roda em loop de polling a cada 2 segundos, lê eventos pendentes da outbox e realiza a entrega HTTP. Rodar como processo independente da API isola falhas nos dois sentidos (crash da API não para notificações; pico no worker não degrada a API de pedidos). Detalhes em [[ADR-002-worker-em-processo-separado-com-polling]].

**3. Entrega resiliente e autenticada.** Cada entrega é assinada com HMAC-SHA256 usando uma secret exclusiva por endpoint de webhook cadastrado (não uma secret global), com suporte a rotação com grace period de 24h ([[ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint]]). Falhas de entrega são reprocessadas com backoff exponencial por até 5 tentativas (1m/5m/30m/2h/12h); eventos que esgotam as tentativas vão para uma Dead Letter Queue (`webhook_dead_letter`), reprocessável manualmente por um administrador ([[ADR-003-politica-de-retry-com-backoff-e-dlq]]). A garantia de entrega é at-least-once, com um `X-Event-Id` único por evento para que o cliente implemente deduplicação ([[ADR-005-garantia-at-least-once-com-x-event-id]]).

O módulo (`src/modules/webhooks/`) segue exatamente a estrutura já usada em `orders`, `users`, `customers` e `products` — controller, service, repository, routes, schemas — e reaproveita `AppError`, o error middleware centralizado, o `requireRole` e o logger Pino já existentes no projeto, sem exigir alterações nesses componentes ([[ADR-006-reuso-de-padroes-existentes-do-projeto]]). O detalhamento de endpoints, contratos, matriz de erros e fluxos passo a passo está no `docs/FDD.md`.

**Limitação conhecida de ordering.** Com um único worker, a entrega de eventos do mesmo pedido preserva a ordem de emissão (por `order_id`), mas não há garantia de ordem global entre pedidos diferentes, nem essa garantia se sustenta caso o processamento seja escalado para múltiplos workers no futuro — registrado explicitamente como limitação a documentar: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker" [09:13, Larissa].

## Alternativas Consideradas

**Disparo síncrono do webhook dentro do `OrderService`.** A alternativa mais simples seria chamar o HTTP client diretamente dentro da transação (ou logo após) de `changeStatus`. Foi descartada porque acoplaria a confiabilidade da transação crítica de negócio à disponibilidade de sistemas de terceiros: "Síncrono não rola... qualquer cliente lento vai travar mudança de status pra outros pedidos" [09:04, Bruno]; "Síncrono está fora de questão" [09:06, Diego]. Trade-off: simplicidade de implementação perdida em troca de isolamento de falhas, considerado não-negociável dado que a transação de pedidos é o núcleo do sistema.

**Redis Streams (ou fila externa dedicada).** Ofereceria desacoplamento nativo e menor latência de detecção de eventos que um MySQL sob polling. Foi descartada por exigir subir e operar infraestrutura adicional desproporcional ao problema e ao tamanho do time: "a gente acabaria precisando subir mais infra... Subir Redis Cluster pra isso é overengineering" [09:07, Larissa/Diego]. Trade-off: menor sofisticação técnica em troca de menor custo operacional e nenhuma infraestrutura nova.

**Trigger de banco de dados / mecanismo tipo `NOTIFY`/`LISTEN`.** Reduziria a latência de detecção de eventos a quase zero, eliminando o polling. Foi descartada porque MySQL não oferece esse mecanismo nativamente (diferente do Postgres), e as alternativas de contorno (escrever em arquivo, chamar um endpoint HTTP a partir do banco) foram julgadas frágeis e desnecessárias: "teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito" [09:09, Diego]. Trade-off: latência de até 2s (ainda dentro da meta de 10s) em troca de uma solução robusta e sem gambiarra.

## Questões em Aberto

- **Rate limiting de envio de webhooks.** Não há, nesta fase, limite de taxa de disparo para um mesmo cliente/endpoint (cenário levantado: 50 pedidos mudando de status no mesmo minuto). A decisão foi observar o comportamento em produção antes de implementar qualquer controle: "Eu acho que não [faz parte do escopo]. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto" [09:38-09:39, Diego/Larissa]. Ação recomendada: instrumentar métricas de volume de envio por cliente desde o início para embasar essa decisão futura.
- **Escalabilidade do worker além de uma única instância.** A decisão atual assume um único worker processando a outbox sequencialmente. Particionamento ou múltiplos workers em paralelo foi adiado explicitamente para quando o volume justificar: "isso é problema do futuro, não agora" [09:13, Diego]. Não há ainda um gatilho quantitativo definido (ex. tamanho de fila, latência observada) para acionar essa evolução — fica como lacuna a ser definida em ciclo futuro.

## Impacto e Riscos

- **Impacto no `OrderService.changeStatus`**: a função passa a ter uma responsabilidade adicional (inserir evento na outbox) dentro da mesma transação. O risco técnico é aumentar a superfície de falha da transação mais crítica do sistema; a mitigação é que a inserção usa o mesmo client transacional (`tx`) já em uso, sem chamada de rede — apenas mais um `INSERT` local ao MySQL.
- **Novo processo de longa duração em produção** (`src/worker.ts`): introduz um novo alvo de operação (restart automático, monitoramento de saúde) que não existia antes.
- **Dependência da confiabilidade de terceiros**: entregas podem falhar por tempo prolongado por causas fora do controle do OMS; mitigado por retry/backoff e DLQ ([[ADR-003-politica-de-retry-com-backoff-e-dlq]]).
- **Risco de segurança de vazamento de secret**: mitigado por secret por endpoint (não global) e suporte a rotação ([[ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint]]).

## Decisões Relacionadas

- [[ADR-001-outbox-pattern-no-mysql]] — Padrão Outbox no MySQL
- [[ADR-002-worker-em-processo-separado-com-polling]] — Worker em processo separado com polling
- [[ADR-003-politica-de-retry-com-backoff-e-dlq]] — Retry com backoff e DLQ
- [[ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint]] — HMAC-SHA256 com secret por endpoint
- [[ADR-005-garantia-at-least-once-com-x-event-id]] — At-least-once com X-Event-Id
- [[ADR-006-reuso-de-padroes-existentes-do-projeto]] — Reuso de padrões existentes
- [[ADR-007-snapshot-do-payload-no-momento-do-evento]] — Snapshot do payload
- [[ADR-008-tls-obrigatorio-e-limite-de-payload]] — TLS obrigatório e limite de payload
