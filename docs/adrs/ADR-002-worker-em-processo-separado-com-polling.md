# ADR-002: Worker em processo separado com polling de 2 segundos

## Status

Aceito

## Contexto

Uma vez que eventos são registrados na tabela `webhook_outbox` (ver [[ADR-001-outbox-pattern-no-mysql]]), algo precisa lê-los e efetivamente entregar as notificações HTTP aos endpoints dos clientes. O requisito de negócio de "tempo real" foi quantificado por Marcos como latência abaixo de 10 segundos entre a mudança de status e a chegada da notificação — [09:02, Marcos].

MySQL não possui um mecanismo nativo de notificação push de eventos (diferente do `NOTIFY/LISTEN` do Postgres), então a leitura da outbox precisa ser feita por consulta periódica (polling) — [09:09, Diego]. Também é preciso decidir onde esse processamento roda: dentro do mesmo processo da API HTTP (`src/server.ts`) ou em um processo separado.

## Decisão

Implementar o processamento como um **processo Node.js separado**, com entry-point próprio (`src/worker.ts`, executado via script `npm run worker`), que roda um loop de **polling a cada 2 segundos**: busca os eventos pendentes mais antigos na `webhook_outbox`, processa a entrega HTTP, e marca o resultado.

> "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca." — [09:09, Diego]

> "Vamos registrar isso como uma decisão. Worker em polling, 2s." — [09:10, Larissa]

O worker separado evita que uma falha ou reinício do processo da API derrube também o processamento de notificações, e vice-versa: "processo separado evita perda do worker em restart da API" — [09:11, Diego]. Cada processo (API e worker) instancia seu próprio `PrismaClient`, apontando para o mesmo `DATABASE_URL` — [09:29-09:30, Diego/Bruno].

O intervalo de 2 segundos é compatível com a meta de latência: mesmo no pior caso (evento inserido logo após o worker iniciar um ciclo), a detecção ocorre em até ~2s, dentro da janela de 10s exigida pelo negócio — [09:02 e 09:09, Marcos/Diego].

## Alternativas Consideradas

**Processar dentro do mesmo processo da API (ex.: via `setInterval` no `server.ts`).** Foi implicitamente descartada ao se decidir por um entry-point próprio análogo ao `src/server.ts` existente — [09:11, Larissa]. Acoplar o worker ao processo da API faria com que um deploy, crash ou pico de carga da API afetasse diretamente a entrega de webhooks (e vice-versa), contrariando a decisão anterior de que a transação de mudança de status não pode depender de infraestrutura externa (ver [09:04, Bruno] em [[ADR-006-reuso-de-padroes-existentes-do-projeto]]).

**Múltiplos workers em paralelo (processamento distribuído/particionado por `order_id`).** Melhoraria a capacidade de throughput em cenários de alto volume, mas exige mecanismo de particionamento ou lock distribuído para evitar entregas duplicadas/concorrência na mesma linha da outbox. Adiada para fase futura por ser prematura frente ao volume atual: "isso é problema do futuro, não agora." — [09:13, Diego]

## Consequências

**Positivas**
- Isolamento de falhas: um crash da API não interrompe o processamento de webhooks pendentes, e um pico no worker não degrada a API de pedidos.
- Deploy e escala independentes dos dois processos.
- Modelo simples (polling) sem dependência de infraestrutura adicional, coerente com a decisão de outbox em MySQL.

**Negativas**
- Com um único worker, o processamento é serializado — não há paralelismo entre eventos processados simultaneamente, o que limita throughput sob alto volume (mitigação futura adiada e registrada como decisão consciente, não esquecimento).
- Ordering de entrega só é garantido por `order_id` e apenas enquanto houver um único worker ativo — [09:12-09:13, Diego/Larissa]; não há garantia de ordem global entre pedidos diferentes.
- Introduz operação de um novo processo de longa duração em produção (novo alvo de monitoramento, restart automático, etc.), que não existia antes na aplicação.
