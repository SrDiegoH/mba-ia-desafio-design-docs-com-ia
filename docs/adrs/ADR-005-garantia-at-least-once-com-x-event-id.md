# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dado o modelo de outbox + worker com retry (ver [[ADR-001-outbox-pattern-no-mysql]] e [[ADR-003-politica-de-retry-com-backoff-e-dlq]]), é possível que o mesmo evento de webhook seja entregue mais de uma vez ao cliente — por exemplo, se a entrega foi bem-sucedida no destino, mas a confirmação (resposta HTTP) não chegou de volta ao worker antes do timeout, levando a uma nova tentativa. É preciso decidir qual garantia de entrega o sistema oferece: exactly-once (cada evento entregue exatamente uma vez) ou at-least-once (cada evento entregue uma ou mais vezes, cabendo ao consumidor lidar com duplicidade).

## Decisão

O sistema garante entrega **at-least-once** (nunca menos, possivelmente mais de uma vez), não exactly-once. Cada evento carrega um identificador único, `X-Event-Id` (UUID gerado no momento da inserção do evento na outbox), permitindo que o cliente implemente deduplicação do seu lado.

> "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos." — [09:25, Diego]

A decisão foi ancorada em práticas de mercado já conhecidas: "Stripe faz assim, GitHub faz assim." — [09:25, Diego]

> "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." — [09:26, Larissa]

## Alternativas Consideradas

**Garantia exactly-once.** Eliminaria a necessidade de deduplicação do lado do cliente, mas exigiria um protocolo de confirmação transacional entre OMS e cada endpoint de cliente (ex.: two-phase commit ou idempotency tokens coordenados nos dois lados), adicionando complexidade significativa de implementação e operação para um ganho marginal, já que o padrão de mercado (Stripe, GitHub) resolve o mesmo problema com at-least-once + id de evento. Descartada por custo/benefício desfavorável: "exigiria coordenação dos dois lados e fica muito mais complexo." — [09:25, Diego]

## Consequências

**Positivas**
- Modelo simples e alinhado a práticas consolidadas de mercado (Stripe, GitHub), o que facilita a integração do lado do cliente — a maioria das equipes de engenharia já conhece o padrão de dedup por id de evento.
- Reduz a complexidade interna do worker: não é necessário confirmar recebimento de forma transacional antes de marcar sucesso, apenas registrar a resposta HTTP obtida.
- O `X-Event-Id` também serve como chave de correlação no histórico de entregas (`GET /webhooks/:id/deliveries`) e em logs/observabilidade.

**Negativas**
- Transfere para o cliente a responsabilidade de implementar deduplicação — se o cliente não implementar corretamente, pode processar o mesmo evento de negócio mais de uma vez (ex. atualizar status duas vezes, ainda que idempotentemente do ponto de vista do domínio dele).
- Não há garantia formal, do lado do OMS, de que a duplicidade nunca ocorrerá — apenas de que a notificação nunca será perdida silenciosamente.
