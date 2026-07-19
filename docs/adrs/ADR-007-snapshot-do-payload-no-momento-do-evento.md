# ADR-007: Snapshot do payload no momento da inserção do evento

## Status

Aceito

## Contexto

Ao inserir um evento na `webhook_outbox` (ver [[ADR-001-outbox-pattern-no-mysql]]), é preciso decidir o que exatamente é armazenado: apenas uma referência ao pedido (`order_id`), a ser resolvida no momento do envio, ou o conteúdo completo do payload já montado no momento em que a mudança de status ocorreu.

Como o worker processa eventos de forma assíncrona e pode haver retries ao longo de várias horas (ver [[ADR-003-politica-de-retry-com-backoff-e-dlq]]), o estado do pedido no banco pode ter mudado entre o momento em que o evento foi gerado e o momento em que ele é efetivamente enviado (ou reenviado).

## Decisão

O evento grava, no momento da inserção na outbox, o **payload já renderizado** (snapshot dos dados relevantes do pedido no instante da mudança de status), e não apenas o `order_id` a ser resolvido depois.

> "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito." — [09:52, Larissa]

Essa decisão foi tomada em conjunto com a definição do formato do payload (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, campos básicos como `total_cents`) — [09:43, Diego] — e do uso de UUID para os identificadores da outbox, seguindo o padrão do projeto — [09:51, Larissa].

## Alternativas Consideradas

**Armazenar apenas `order_id` e resolver o payload no momento do envio (lazy).** Reduziria o tamanho da linha na outbox e manteria o evento sempre "atualizado" com o estado mais recente do pedido no momento do envio. Foi descartada porque um evento reenviado (após retry) ou processado com atraso poderia carregar um snapshot de status diferente do que efetivamente motivou o evento — por exemplo, um evento de `PAID` sendo enviado horas depois já mostrando o pedido como `CANCELLED`, criando incoerência entre o `event_type`/`to_status` do payload e o estado real consultado em tempo real. A discussão em [09:51-09:52, Larissa/Diego/Bruno] tratou esse "caso esquisito" como motivo suficiente para preferir o snapshot.

## Consequências

**Positivas**
- Cada evento entregue é internamente consistente: `to_status` no payload sempre corresponde exatamente à transição que gerou aquele evento específico, independentemente de quanto tempo se passou até a entrega (ou reentrega).
- Simplifica o worker: não é necessário fazer um `SELECT` adicional no pedido no momento do envio, reduzindo carga no banco e pontos de falha durante o processamento.
- Elimina condições de corrida entre múltiplas mudanças de status rápidas do mesmo pedido gerando eventos com dados inconsistentes entre si.

**Negativas**
- Aumenta o tamanho médio das linhas da `webhook_outbox` (payload completo por evento, em vez de apenas uma referência).
- Se o cliente precisar de dados adicionais do pedido não incluídos no snapshot (o payload é intencionalmente enxuto, sem `items` — [09:43, Diego]), ele precisa consultar `GET /orders/:id` separadamente — decisão consciente para não inflar o payload de webhook.
