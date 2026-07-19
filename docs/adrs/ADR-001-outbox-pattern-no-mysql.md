# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito

## Contexto

O OMS precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de pedido em tempo real, hoje feito via polling manual em `GET /orders` [09:00, Marcos]. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179), dentro de uma transação Prisma (`this.prisma.$transaction`) que atualiza `Order`, ajusta estoque e grava auditoria em `OrderStatusHistory`.

O desafio central é garantir consistência entre "o status mudou no banco" e "o evento de notificação foi registrado para envio". Se essas duas operações não forem atômicas, é possível a transação de negócio commitar sem o evento ser registrado (perda de notificação) ou o evento ser registrado sem a transação principal ter sido efetivada (evento fantasma).

## Decisão

Adotar o padrão **Transactional Outbox** usando a própria tabela do MySQL: uma nova tabela `webhook_outbox` recebe uma linha dentro da **mesma transação SQL** que atualiza `orders` e insere em `order_status_history`. Um worker separado lê essa tabela de forma assíncrona e realiza a entrega HTTP.

> "Quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. Se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível." — [09:06, Diego]

Decisão confirmada por Larissa: "Tá decidido então: outbox em MySQL." — [09:08, Larissa]

Do ponto de vista de implementação, a inserção na outbox é feita através de uma função (`publishWebhookEvent(tx, order, fromStatus, toStatus)`) que recebe o mesmo client transacional (`tx`) já usado pelo `changeStatus`, e falha (rollback) em conjunto se a inserção falhar — [09:40-09:41, Bruno/Diego].

## Alternativas Consideradas

**Redis Streams / fila externa dedicada.** Proporia baixa latência e desacoplamento nativo de fila, mas exigiria subir e operar mais um componente de infraestrutura (cluster Redis) só para esse propósito. Descartada por overengineering dado o tamanho do time: "A alternativa seria botar Redis Streams... mas a gente acabaria precisando subir mais infra... Subir Redis Cluster pra isso é overengineering." — [09:07, Larissa/Diego]

**Trigger de banco de dados / mecanismo tipo NOTIFY-LISTEN.** MySQL não oferece um mecanismo nativo de notificação de eventos como o `NOTIFY/LISTEN` do Postgres, o que forçaria soluções improvisadas (escrever em arquivo, chamar um endpoint HTTP a partir do banco). Descartada por complexidade desnecessária frente ao requisito de latência (<10s): "MySQL não tem listener nativo... Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito." — [09:09, Diego]

## Consequências

**Positivas**
- Atomicidade garantida sem coordenação distribuída: a mudança de status e o registro do evento vivem/morrem juntos na mesma transação MySQL.
- Nenhuma infraestrutura nova: reaproveita o mesmo banco MySQL e o mesmo `PrismaClient`/schema já usados pelo restante do OMS.
- Modelo simples de entender e operar para um time pequeno, alinhado ao princípio de reuso de padrões existentes ([[ADR-006-reuso-de-padroes-existentes-do-projeto]]).

**Negativas**
- A tabela `webhook_outbox` precisa ser lida por polling (não há push nativo do MySQL), o que introduz latência mínima de detecção (mitigada pelo intervalo de 2s definido em [[ADR-002-worker-em-processo-separado-com-polling]]).
- Escalabilidade horizontal do processamento é limitada por não haver, nesta fase, particionamento entre múltiplos workers (explicitamente adiado — [09:13, Diego]).
- Requer atenção a índices (`status`, `created_at`) e a uma futura política de arquivamento/limpeza de linhas entregues (mencionada como fora de escopo desta feature — [09:08, Diego]) para a tabela não crescer indefinidamente.
