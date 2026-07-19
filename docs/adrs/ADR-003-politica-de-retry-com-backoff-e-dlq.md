# ADR-003: Política de retry com backoff exponencial e Dead Letter Queue

## Status

Aceito

## Contexto

Endpoints de webhook são controlados pelos clientes (Atlas, MaxDistribuição, Nova Cargo) e podem estar indisponíveis por período variável — de instabilidades curtas a manutenções planejadas de horas. O time já observou um caso real de indisponibilidade de cliente por duas horas em manutenção planejada — [09:16, Diego]. É preciso decidir quantas tentativas de reenvio realizar, com qual espaçamento, e o que fazer com eventos que esgotam as tentativas sem sucesso.

## Decisão

Adotar **5 tentativas de entrega**, com backoff crescente na seguinte progressão: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** (janela total de cobertura de aproximadamente 15 horas).

> "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas." — [09:17, Diego]

> "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h." — [09:17, Larissa]

Eventos que esgotam as 5 tentativas são movidos para uma tabela separada, `webhook_dead_letter`, contendo o payload, o motivo da última falha e o timestamp — [09:18, Diego]. Racional para a tabela separada: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento." — [09:18, Diego]

Eventos na DLQ podem ser reprocessados manualmente através de um endpoint administrativo, `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na `webhook_outbox` como pendente — [09:18, Diego]. Esse endpoint exige role `ADMIN` (reaproveitando o middleware `requireRole` existente) e gera log de auditoria de quem executou o replay — [09:35-09:36, Sofia/Larissa].

## Alternativas Consideradas

**Retry fixo de 3 tentativas.** Foi a primeira proposta em discussão, mas descartada por não cobrir janelas de indisponibilidade realistas de clientes: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada." — [09:16, Diego]

**Retry indefinido com backoff crescente (sem limite de tentativas).** Evitaria a necessidade de uma DLQ, mas gera o risco de eventos "pendurados" indefinidamente para clientes que nunca mais respondem, poluindo a fila ativa de processamento. Descartada: "traz o problema de evento ficar pendurado pra sempre se o cliente sumiu." — [09:15, Diego]

## Consequências

**Positivas**
- Janela de retry (~15h) cobre indisponibilidades reais já observadas em clientes (ex.: manutenção de 2h), reduzindo falsa perda de notificação.
- DLQ separada mantém a tabela `webhook_outbox` operacional enxuta (só eventos ativos/pendentes), sem misturar eventos "mortos" com os que ainda estão em processamento.
- Reprocessamento manual via endpoint dá controle operacional para casos excepcionais, com trilha de auditoria de quem interveio.

**Negativas**
- Eventos que falham definitivamente exigem intervenção manual (não há retry automático após a DLQ) — depende de monitoramento humano ou de alertas para não passarem despercebidos (não coberto nesta fase; ver observabilidade no FDD).
- A garantia é at-least-once, não exactly-once (ver [[ADR-005-garantia-at-least-once-com-x-event-id]]): um evento pode, em tese, ser entregue e mesmo assim marcado como falho por timeout de resposta, levando a nova tentativa e possível duplicidade — mitigado pelo `X-Event-Id`.
- Backoff de até 12h entre tentativas significa que, no pior caso, uma notificação pode levar até ~15h para ser confirmada como falha definitiva, exigindo que o cliente trate isso operacionalmente (ex. consultando `GET /webhooks/:id/deliveries`).
