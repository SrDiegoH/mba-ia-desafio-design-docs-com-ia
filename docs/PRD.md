# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto

O OMS opera hoje sem qualquer mecanismo de notificação externa: clientes B2B que precisam saber quando o status de um pedido muda fazem polling manual contra `GET /orders` [09:00, Marcos]. Esta feature introduz um sistema de webhooks que notifica clientes automaticamente, em tempo real, sempre que o status de um pedido deles muda, eliminando a necessidade de polling.

## Problema e Motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente notificação em tempo real de mudança de status de pedidos [09:00, Marcos]. O mecanismo atual (polling manual) não atende a essa necessidade de forma satisfatória. O risco de negócio é concreto: "A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente" [09:00, Marcos]. A ausência de um mecanismo de push também é um limitador para a plataforma como um todo — qualquer integração B2B futura enfrentaria a mesma lacuna.

## Público-Alvo e Cenários de Uso

**Público-alvo primário**: clientes B2B da plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo) que integram sistemas próprios ao OMS e precisam reagir a mudanças de status de pedido sem depender de polling.

**Cenário principal**: um pedido de um cliente B2B muda de status (ex. `PROCESSING` → `SHIPPED`). O sistema do cliente, previamente cadastrado com um endpoint de webhook, recebe uma notificação HTTP autenticada em até 10 segundos, contendo os dados essenciais da mudança, e pode agir sobre ela (ex. disparar rastreamento de entrega, atualizar seu próprio painel).

**Cenário secundário (operação interna)**: um administrador do OMS precisa investigar e reprocessar manualmente notificações que falharam definitivamente após todas as tentativas automáticas (via consulta à Dead Letter Queue e endpoint de replay).

## Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a necessidade de polling para detectar mudança de status | Latência entre mudança de status e entrega da notificação | < 10 segundos, para endpoints saudáveis — [09:02, Marcos] |
| Garantir confiabilidade de entrega mesmo com instabilidade do cliente | Cobertura de retry antes de mover para DLQ | Até 5 tentativas, cobrindo até ~15h de indisponibilidade do cliente — [09:17, Diego] |
| Reduzir risco de churn dos clientes B2B que solicitaram a feature | Entrega da feature dentro do prazo de negócio | Fim do trimestre (prazo citado pela Atlas) — [09:00 e 09:45, Marcos] |
| Autenticidade e integridade das notificações | Cobertura de assinatura HMAC | 100% das entregas assinadas com HMAC-SHA256 — [09:20, Sofia] |

## Escopo

**Incluso:**
- CRUD de configuração de webhook por cliente (URL, secret, filtro de status) — [09:31-09:33, Marcos/Bruno]
- Publicação de evento na mesma transação da mudança de status (outbox) — [09:06, Diego]
- Worker assíncrono com retry/backoff e Dead Letter Queue — [09:09-09:18, Diego]
- Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h — [09:20-09:22, Sofia]
- Garantia de entrega at-least-once com `X-Event-Id` para deduplicação no cliente — [09:25-09:26, Diego]
- Histórico de entregas (`GET /webhooks/:id/deliveries`) — [09:34-09:35, Marcos]
- Endpoint administrativo de replay de eventos da DLQ, restrito a role `ADMIN` — [09:18 e 09:35-09:36, Diego/Sofia]
- TLS obrigatório e limite de 64KB por payload, com falha explícita — [09:23-09:24, Sofia]

**Fora de Escopo:**
- **Notificação por e-mail em caso de falha repetida de entrega.** Explicitamente adiada: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" — [09:37-09:38, Marcos/Larissa].
- **Dashboard visual para o cliente gerenciar webhooks.** Adiado, tratado como projeto separado do time de frontend: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" — [09:39-09:40, Marcos/Larissa].
- **Múltiplos workers em paralelo / processamento particionado.** Adiado explicitamente: "isso é problema do futuro, não agora" — [09:13, Diego].
- **Rate limiting de envio de webhooks.** Deixado como questão em aberto, a ser observada em produção antes de decidir implementação — [09:38-09:39, Diego/Larissa].
- **Arquivamento/purga de eventos entregues da outbox após 30 dias.** Mencionado como necessário eventualmente, mas fora do escopo desta feature — [09:08, Diego].
- **Endurecimento de permissões do CRUD de configuração de webhook** (hoje qualquer usuário autenticado pode gerenciar). Adiado: "Por enquanto sim. Mais pra frente a gente pode endurecer" — [09:37, Sofia].

## Requisitos Funcionais

1. Cliente pode cadastrar um webhook informando `url`, lista de status desejados e `customer_id`; o sistema gera e retorna a `secret` na criação — [09:31, Marcos].
2. O sistema permite editar (`PATCH`) uma configuração de webhook existente — [09:33, Bruno].
3. O sistema permite remover (`DELETE`) uma configuração de webhook — [09:33, Bruno].
4. O sistema permite listar (`GET`) as configurações de webhook de um cliente — [09:33, Bruno].
5. O cliente pode filtrar quais status de pedido deseja receber notificação (ex. apenas `SHIPPED` e `DELIVERED`) — [09:33-09:34, Marcos].
6. O sistema disponibiliza histórico das últimas 100 entregas de um webhook, incluindo sucesso/falha, payload, resposta e tempo de resposta — [09:34-09:35, Marcos].
7. Um administrador pode reprocessar manualmente um evento que caiu na Dead Letter Queue — [09:18 e 09:35, Diego].
8. O cliente pode rotacionar a secret de um webhook, com a secret anterior permanecendo válida por 24h (grace period) — [09:21, Sofia].
9. O sistema insere o evento de notificação na mesma transação de banco que efetiva a mudança de status do pedido — [09:06 e 09:40-09:41, Diego/Bruno].
10. O sistema entrega notificações assinadas com HMAC-SHA256, permitindo ao cliente verificar autenticidade e integridade — [09:20, Sofia].
11. O sistema reenvia automaticamente notificações que falharam, seguindo uma política de backoff, antes de desistir — [09:15-09:17, Diego].

## Requisitos Não Funcionais

- **Latência**: notificação entregue em menos de 10 segundos após a mudança de status, para endpoints saudáveis — [09:02, Marcos].
- **Disponibilidade/isolamento**: a transação de mudança de status não pode ser bloqueada ou comprometida por indisponibilidade de sistemas de terceiros — [09:04, Bruno].
- **Segurança de transporte**: URLs de webhook devem usar HTTPS — [09:23, Sofia].
- **Segurança de autenticidade**: assinatura HMAC-SHA256 obrigatória em toda entrega — [09:19-09:20, Sofia].
- **Segurança de segredo**: secret exclusiva por endpoint, nunca global — [09:21, Sofia].
- **Ordering (limitação conhecida)**: entregas de eventos do mesmo pedido preservam a ordem de emissão (por `order_id`), mas não há garantia de ordem global entre pedidos diferentes, e essa garantia depende de o processamento continuar sendo feito por um único worker — [09:12-09:13, Diego/Larissa].
- **Limite de payload**: 64KB por evento, com falha explícita (não truncamento) acima do limite — [09:23-09:24, Sofia].
- **Timeout de entrega**: 10 segundos por tentativa HTTP do worker — [09:42, Diego].
- **Idempotência**: cada evento carrega identificador único (`X-Event-Id`) para permitir deduplicação do lado do cliente — [09:24-09:25, Diego].
- **Auditabilidade**: toda ação de replay de evento da DLQ é registrada com identidade de quem executou — [09:36, Sofia].

## Decisões e Trade-offs Principais

- **Outbox em MySQL em vez de fila externa (Redis Streams)**: evita infraestrutura adicional, ao custo de depender de polling em vez de push nativo — ver [[adrs/ADR-001-outbox-pattern-no-mysql]].
- **Processamento assíncrono (worker separado) em vez de disparo síncrono**: adiciona um componente novo em produção, mas evita acoplar a confiabilidade da transação de pedidos à disponibilidade de terceiros — ver [[adrs/ADR-002-worker-em-processo-separado-com-polling]].
- **At-least-once em vez de exactly-once**: mais simples de implementar e operar, mas exige que o cliente implemente deduplicação — ver [[adrs/ADR-005-garantia-at-least-once-com-x-event-id]].
- **5 tentativas com backoff de até 12h em vez de 3 tentativas rápidas**: cobre indisponibilidades reais de clientes, ao custo de eventos "pendentes" por até ~15h antes de ir para DLQ — ver [[adrs/ADR-003-politica-de-retry-com-backoff-e-dlq]].

Detalhamento completo das decisões técnicas em `docs/RFC.md` e `docs/adrs/`.

## Dependências

- Banco MySQL já existente (via Prisma) — nenhuma infraestrutura nova requerida.
- Estrutura de módulos, classes de erro, middleware de autenticação/autorização e logger já existentes no OMS (ver `docs/FDD.md`, seção "Integração com o sistema existente").
- Disponibilidade da equipe de segurança (Sofia) para revisão dedicada antes do lançamento — pelo menos 2 dias úteis reservados — [09:46, Sofia].

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente com indisponibilidade prolongada (ex. manutenção planejada) causa perda de notificações | Média — já ocorreu antes com cliente real (indisponibilidade de 2h) [09:16, Diego] | Alto — pedido não notificado pode gerar decisão de negócio errada no cliente | Retry com backoff de até ~15h cobrindo esse cenário; eventos esgotados vão para DLQ (não são descartados) e podem ser reprocessados manualmente — [[adrs/ADR-003-politica-de-retry-com-backoff-e-dlq]] |
| Vazamento de secret de um cliente compromete autenticidade das notificações para ele | Média — já ocorreu antes (vazamento em log de aplicação do cliente) [09:22, Diego] | Médio — afeta apenas o cliente cuja secret vazou, não a plataforma inteira | Secret exclusiva por endpoint (não global) + suporte a rotação com grace period de 24h — [[adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint]] |
| Não entrega da feature dentro do prazo de negócio (fim do trimestre) leva a churn de cliente estratégico | Baixa a Média — prazo apertado (3 sprints estimadas) [09:45-09:46, Marcos/Larissa] | Alto — perda de receita/relacionamento com cliente Atlas | Escopo desta fase deliberadamente restrito (dashboard, e-mail e multi-worker adiados) para viabilizar entrega dentro do prazo |
| Nova responsabilidade dentro da transação `changeStatus` degrada performance ou introduz nova fonte de falha na transação crítica de pedidos | Baixa | Alto — pedidos são o núcleo do sistema | Operação adicional é um `INSERT` local à mesma transação MySQL (sem chamada de rede); medir impacto em teste de carga antes do rollout |

## Critérios de Aceitação

- Cliente consegue cadastrar, editar, listar e remover configurações de webhook via API.
- Uma mudança de status de pedido com filtro correspondente gera notificação entregue em até 10s para endpoint saudável.
- Notificação enviada contém `X-Signature` válida e `X-Event-Id` estável entre tentativas.
- Falha de entrega é reprocessada automaticamente conforme a política de backoff, e eventos esgotados aparecem na Dead Letter Queue.
- Administrador consegue reprocessar manualmente um evento da DLQ, e a ação fica registrada em log de auditoria.
- Cliente consegue rotacionar a secret de um webhook sem interrupção de entregas (grace period de 24h respeitado).
- Histórico de entregas (`deliveries`) reflete corretamente sucesso/falha, status HTTP e tempo de resposta das últimas 100 tentativas.

## Estratégia de Testes e Validação

- **Testes de integração** cobrindo a transação de `changeStatus` estendida: confirmar que rollback da transação principal também reverte a inserção do evento na outbox (atomicidade).
- **Testes do worker**: simulação de endpoint indisponível/lento validando a progressão de backoff (1m/5m/30m/2h/12h) e a movimentação correta para DLQ após a 5ª tentativa.
- **Testes de contrato dos endpoints HTTP** (CRUD de webhook, deliveries, replay de DLQ), incluindo os cenários da matriz de erros `WEBHOOK_*` (ver `docs/FDD.md`).
- **Testes de segurança**: verificação de que a assinatura HMAC enviada é válida e recalculável pelo cliente; rejeição de URLs não-HTTPS; rejeição de payload acima de 64KB.
- **Revisão de segurança dedicada** pela equipe de Sofia antes do lançamento, com pelo menos 2 dias úteis reservados no cronograma — [09:46, Sofia].
- **Teste de carga** no fluxo de `changeStatus` para validar que a inserção adicional na outbox não degrada a performance da transação de pedidos de forma perceptível.
