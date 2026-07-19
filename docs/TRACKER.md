# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos de design à sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte do OMS.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Ausência de mecanismo de notificação externa; cliente usa polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Risco de churn do cliente Atlas se feature não sair até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega < 10 segundos define "tempo real" | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (url, statuses, customer_id), secret retornada na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição (PATCH) de configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção (DELETE) de configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem (GET) de webhooks por cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status desejado | TRANSCRICAO | [09:33-09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de últimas 100 entregas (deliveries) | TRANSCRICAO | [09:34-09:35] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay administrativo de evento da DLQ | TRANSCRICAO | [09:18][09:35] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Inserção do evento na mesma transação do changeStatus | TRANSCRICAO | [09:06][09:40-09:41] Diego/Bruno |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Entrega assinada com HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reenvio automático de notificações falhas via backoff | TRANSCRICAO | [09:15-09:17] Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Transação de status não pode ser bloqueada por chamada a terceiros | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório nas URLs de webhook | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB com falha explícita | TRANSCRICAO | [09:23-09:24] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa HTTP do worker | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | X-Event-Id para idempotência/deduplicação no cliente | TRANSCRICAO | [09:24-09:25] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Auditoria de quem executa replay de DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-RNF-08 | docs/PRD.md | Restrição | Ordering: garantia só por order_id, sem ordem global, apenas single-worker | TRANSCRICAO | [09:12-09:13] Diego/Larissa |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição | Fora de escopo: notificação por e-mail em falha repetida | TRANSCRICAO | [09:37-09:38] Marcos/Larissa |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual de gestão de webhooks | TRANSCRICAO | [09:39-09:40] Marcos/Larissa |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição | Fora de escopo: múltiplos workers em paralelo | TRANSCRICAO | [09:13] Diego |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de envio (questão em aberto) | TRANSCRICAO | [09:38-09:39] Diego/Larissa |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de outbox após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-06 | docs/PRD.md | Restrição | Fora de escopo: endurecer permissões do CRUD de webhook | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-01 | docs/PRD.md | Trade-off | Risco de indisponibilidade prolongada de cliente, mitigado por retry+DLQ | TRANSCRICAO | [09:16] Diego |
| PRD-RISCO-02 | docs/PRD.md | Trade-off | Risco de vazamento de secret, mitigado por secret por endpoint + rotação | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-03 | docs/PRD.md | Trade-off | Risco de não cumprir prazo de negócio (fim de trimestre) | TRANSCRICAO | [09:45-09:46] Marcos/Larissa |
| PRD-RISCO-04 | docs/PRD.md | Trade-off | Risco de degradar performance da transação changeStatus | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança dedicada antes do lançamento (min. 2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox pattern em MySQL como mecanismo de registro de eventos | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado em polling de 2s como mecanismo de processamento | TRANSCRICAO | [09:09-09:10] Diego/Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Entrega autenticada, retry e at-least-once como estratégia de resiliência | TRANSCRICAO | [09:17][09:22][09:25] Diego/Sofia |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa descartada: disparo síncrono no OrderService | TRANSCRICAO | [09:04][09:06] Bruno/Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa descartada: Redis Streams (overengineering) | TRANSCRICAO | [09:07] Larissa/Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Alternativa descartada: trigger de banco tipo NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de envio de webhooks | TRANSCRICAO | [09:38-09:39] Diego/Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: escalabilidade do worker além de uma instância | TRANSCRICAO | [09:13] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Risco | Impacto no changeStatus por responsabilidade adicional na transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Novo processo de longa duração em produção (worker) | CODIGO | src/server.ts |
| RFC-RESTR-01 | docs/RFC.md | Restrição | Limitação conhecida de ordering: só por order_id, apenas single-worker | TRANSCRICAO | [09:13] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Requisito Funcional | Fluxo de criação do evento na outbox dentro de changeStatus | TRANSCRICAO | [09:06][09:40-09:41] Diego/Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Requisito Funcional | Fluxo de processamento pelo worker (polling, envio, marcação) | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Tabela de backoff (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Requisito Funcional | Fluxo de DLQ e replay administrativo | TRANSCRICAO | [09:18][09:35-09:36] Diego/Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | Endpoint POST /webhooks (criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | Endpoint PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | Endpoint DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | Endpoint GET /webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | Endpoint POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | Endpoint GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34-09:35] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | Endpoint POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18][09:35] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Decisão | Formato do payload de evento (campos e headers) | TRANSCRICAO | [09:43-09:45] Diego/Sofia |
| FDD-ERRO-01 | docs/FDD.md | Decisão | Padrão de código de erro WEBHOOK_* espelhando InsufficientStockError | TRANSCRICAO | [09:28-09:29] Bruno/Larissa |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por tentativa | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Falha explícita em payload > 64KB (sem truncar) | TRANSCRICAO | [09:23] Sofia |
| FDD-OBS-01 | docs/FDD.md | Decisão | Reuso do logger Pino com convenção de eventos snake_case | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Ponto de integração: método changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Decisão | Reuso da hierarquia de classes de erro AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-03 | docs/FDD.md | Decisão | Reuso do error middleware centralizado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | Reuso do middleware authenticate/requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Decisão | Reuso do logger Pino singleton | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | Novos models no schema Prisma seguindo padrão UUID existente | CODIGO | prisma/schema.prisma |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Registro de rotas seguindo padrão de composição por módulo | CODIGO | src/routes/index.ts |
| FDD-INTEG-08 | docs/FDD.md | Decisão | Novas env vars seguindo padrão Zod de validação no boot | CODIGO | src/config/env.ts |
| FDD-INTEG-09 | docs/FDD.md | Restrição | NotFoundError precisa aceitar código customizável para emitir WEBHOOK_NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-OBS-02 | docs/FDD.md | Restrição | redact.paths do Pino não cobre *.secret; requer ajuste antes do lançamento | CODIGO | src/shared/logger/index.ts |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Outbox pattern em MySQL, mesma transação do changeStatus | TRANSCRICAO | [09:06-09:08] Diego/Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Trade-off | Alternativas descartadas: Redis Streams e trigger de banco | TRANSCRICAO | [09:07][09:09] Larissa/Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado, polling de 2s | TRANSCRICAO | [09:09-09:11] Diego/Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Alternativa descartada: múltiplos workers paralelos (adiado) | TRANSCRICAO | [09:13] Diego |
| ADR-003-DEC | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Decisão | Retry de 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:15-09:18] Diego/Larissa |
| ADR-003-ALT | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Trade-off | Alternativas descartadas: 3 tentativas e retry indefinido | TRANSCRICAO | [09:15-09:16] Diego |
| ADR-004-DEC | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace period 24h | TRANSCRICAO | [09:20-09:22] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa descartada: secret global compartilhada | TRANSCRICAO | [09:21] Sofia |
| ADR-005-DEC | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:25-09:26] Diego/Larissa |
| ADR-005-ALT | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Alternativa descartada: garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md | Decisão | Reuso máximo de padrões existentes (módulos, erros, logger, auth) | TRANSCRICAO | [09:27-09:30] Bruno/Larissa |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md | Restrição | Estrutura de módulo padrão a seguir | CODIGO | src/modules/orders |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md | Restrição | Hierarquia de erros a espelhar | CODIGO | src/shared/errors/app-error.ts |
| ADR-007-DEC | docs/adrs/ADR-007-snapshot-do-payload-no-momento-do-evento.md | Decisão | Payload salvo como snapshot no momento do evento | TRANSCRICAO | [09:51-09:52] Larissa/Diego/Bruno |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-no-momento-do-evento.md | Trade-off | Alternativa descartada: resolver payload em lazy no envio | TRANSCRICAO | [09:51-09:52] Larissa/Diego/Bruno |
| ADR-008-DEC | docs/adrs/ADR-008-tls-obrigatorio-e-limite-de-payload.md | Decisão | TLS obrigatório e limite de 64KB com falha explícita | TRANSCRICAO | [09:23-09:24] Sofia/Diego/Larissa |
| ADR-008-ALT | docs/adrs/ADR-008-tls-obrigatorio-e-limite-de-payload.md | Trade-off | Alternativas descartadas: permitir HTTP e truncar payload | TRANSCRICAO | [09:23] Sofia |
