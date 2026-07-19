**NOTA:** Preferi deixar o conteúdo original do README.md + Novo conteúdo solicitado no desafio.

# Da Reunião ao Documento: Design Docs Gerados por IA  | Conteúdo original do desafio

## Descrição

Neste desafio você vai transformar a transcrição de uma reunião técnica em um pacote completo de design docs, usando IA como ferramenta principal de produção.

**Cenário:** uma empresa que opera um Order Management System (OMS) em produção vai construir uma nova feature, um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição da call (`TRANSCRICAO.md`).

**Sua tarefa:** produzir, a partir da transcrição e do código existente, a documentação técnica da feature, em nível acionável o suficiente para o time de engenharia iniciar a implementação.

## Sobre o uso de IA

A IA é sua ferramenta principal de produção neste desafio. Você vai usá-la para ler o código, analisar a transcrição, estruturar os documentos e gerar o conteúdo final. O que se espera de você é o papel de maestro: definir o que precisa ser feito, formular bons prompts, revisar criticamente o que a IA entrega, corrigir e refinar até o resultado ficar consistente.

## Estrutura do desafio

O desafio consiste em produzir um **pacote de design docs**: PRD, RFC, FDD, ADRs, Tracker e o README do processo a partir da transcrição e do código.

## Objetivo

Entregar, em um repositório público no GitHub (fork do repositório base), o seguinte pacote de documentação:

- PRD (Product Requirement Document) da feature
- RFC (Request for Comments) com a proposta técnica da solução, submetida à equipe para revisão
- FDD (Feature Design Document) da feature
- Entre 5 e 8 ADRs (Architecture Decision Records) das decisões discutidas
- Tracker de rastreabilidade ligando cada item à origem na transcrição ou no código
- README atualizado documentando o processo de produção

Toda informação registrada nos documentos deve ser rastreável à transcrição ou ao código fonte da aplicação. Não é permitido inventar requisitos, decisões ou restrições sem origem identificável.

### O pacote de documentos e o papel de cada um

Os documentos não se repetem: cada um opera em uma **altura** diferente. Antes de produzir, entenda a fronteira entre eles: conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

| Documento | Papel | Altura | Pergunta que responde |
| --- | --- | --- | --- |
| **PRD** | Problema, público, escopo e métricas de sucesso | Produto / negócio | *Por que e o quê?* |
| **RFC** | Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto | Arquitetura | *Como pretendemos resolver, e o que ainda está em aberto?* |
| **ADRs** | Cada decisão arquitetural isolada, com contexto e consequências | Decisão pontual | *Por que decidimos exatamente assim?* |
| **FDD** | Especificação de implementação: fluxos, contratos, erros, integração com o código | Implementação | *Como construir, em detalhe?* |
| **Tracker** | Rastreabilidade de cada item ao código ou à transcrição | Transversal | *De onde veio cada coisa?* |

Em uma frase: o **RFC propõe e abre para revisão**, os **ADRs registram cada decisão fechada** e o **FDD detalha como construir**. O RFC é conciso (2 a 4 páginas) e fala em decisão; o FDD é profundo e fala em implementação. Não repita no RFC o nível de detalhe do FDD.

## Contexto

### A aplicação existente

O repositório base contém uma aplicação Node.js + TypeScript funcional: um Order Management System com módulos de autenticação, usuários, clientes, produtos e pedidos. Banco MySQL via Prisma. O ciclo de vida do pedido tem máquina de estados controlada, controle transacional de estoque e auditoria de mudanças de status.

A aplicação não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Esse vácuo é proposital. É exatamente o que a feature discutida na reunião pretende preencher.

Seus documentos vão precisar referenciar componentes do código existente, como a estrutura modular, a máquina de estados, a transação do `changeStatus`, as classes de erro, o padrão de códigos de erro, o middleware `requireRole`, o error middleware centralizado e o logger Pino. Use a IA para mapear esses pontos a partir do código.

### A transcrição

O arquivo `TRANSCRICAO.md` contém a gravação literal da reunião técnica. Cinco participantes discutem por aproximadamente 55 minutos no formato `[hh:mm] Nome: fala`.

A transcrição inclui decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o código existente, pontos descartados ou adiados para fases futuras e detalhes técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO entra é tão importante quanto identificar o que entra. Use a IA com prompts dirigidos para fazer essa filtragem, não pedidos genéricos.

## Tecnologias e ferramentas

Liberdade total na escolha de ferramentas de IA. Você pode usar qualquer combinação de Claude, ChatGPT, Cursor, Copilot Chat, Gemini, agentes, prompts customizados, skills ou plugins. Aproveite os prompts e plugins disponibilizados pelo professor durante o curso como ponto de partida.

Os documentos devem ser entregues em formato Markdown.

A entrega é puramente documental: você não deve mexer no código da aplicação (`src/`, `prisma/`, `tests/`, configurações). O código serve de contexto e referência.

## Requisitos

### 1. PRD da feature

Produza o arquivo `docs/PRD.md` cobrindo a feature de Sistema de Webhooks de Notificação de Pedidos. O PRD deve seguir o formato apresentado no curso e incluir, no mínimo, as seguintes seções:

- Resumo e contexto da feature
- Problema e motivação
- Público-alvo e cenários de uso
- Objetivos e métricas de sucesso
- Escopo (incluso e fora de escopo)
- Requisitos funcionais
- Requisitos não funcionais
- Decisões e trade-offs principais
- Dependências
- Riscos e mitigação
- Critérios de aceitação
- Estratégia de testes e validação

A seção "Fora de escopo" deve listar explicitamente pelo menos 2 itens descartados ou adiados durante a reunião.

### 2. RFC da feature

Produza o arquivo `docs/RFC.md` com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão. O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. É um documento conciso (2 a 4 páginas); o detalhamento de implementação fica no FDD. Deve seguir o formato apresentado no curso e incluir, no mínimo:

- Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
- Resumo executivo (TL;DR) da proposta
- Contexto e problema
- Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
- Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
- Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
- Impacto e riscos
- Decisões relacionadas (links para os ADRs correspondentes)

O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD.

### 3. FDD da feature

Produza o arquivo `docs/FDD.md` detalhando o "como implementar" da feature. O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar. Deve seguir o formato apresentado no curso e incluir, no mínimo:

- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões
- Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
- Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
- Matriz de erros previstos com códigos no padrão `WEBHOOK_*`
- Estratégias de resiliência (timeouts, retries, backoff, fallback)
- Observabilidade (métricas, logs, tracing)
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação

Seção obrigatória adicional, específica deste desafio: **"Integração com o sistema existente"**. Esta seção deve nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um (por exemplo, como o método `changeStatus` será estendido, como as classes de erro existentes serão reutilizadas).

### 4. ADRs

Produza entre 5 e 8 ADRs em arquivos separados dentro de `docs/adrs/`, nomeados no formato `ADR-NNN-titulo-em-kebab-case.md` (ex: `ADR-001-outbox-no-mysql.md`).

Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:

- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com `X-Event-Id`
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto

Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.

### 5. Tracker de Rastreabilidade

Produza o arquivo `docs/TRACKER.md`, uma tabela markdown que mapeia cada item registrado nos seus documentos à origem na transcrição ou no código. O tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

O tracker não é um conceito padrão do mercado nem é um documento abordado diretamente no curso. É uma exigência específica deste desafio que ajuda a manter a integridade da documentação contra alucinações da IA.

Formato obrigatório da tabela:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |

Onde:

- **ID**: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
- **Documento**: arquivo onde o item aparece (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-...md`)
- **Tipo**: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
- **Conteúdo (resumo)**: descrição de uma linha do item
- **Fonte**: `TRANSCRICAO` ou `CODIGO`
- **Localização**: para `TRANSCRICAO`, timestamp + nome do falante (ex: `[09:17] Diego`). Para `CODIGO`, caminho do arquivo (ex: `src/modules/orders/order.service.ts`).

Cobertura mínima: pelo menos 80% dos itens identificáveis nos seus documentos devem ter linha correspondente no tracker.

### 6. README com o processo

O `README.md` na raiz do repositório base contém este enunciado. Substitua o conteúdo dele pela documentação do seu processo de produção. Você pode manter um link ou seção fazendo referência ao enunciado original se quiser, mas o foco do novo conteúdo é descrever sua jornada.

Estrutura obrigatória do novo README:

- **Sobre o desafio**: 1 a 2 parágrafos descrevendo a tarefa em suas palavras
- **Ferramentas de IA utilizadas**: lista das ferramentas que você usou, com breve nota sobre o papel de cada uma
- **Workflow adotado**: como você organizou o trabalho. Em que ordem produziu os documentos, como organizou a interação com a IA
- **Prompts customizados**: pelo menos 2 prompts relevantes que você escreveu ou adaptou, mostrados em blocos de código
- **Iterações e ajustes**: descreva os principais momentos em que a IA gerou algo errado ou superficial e você teve que corrigir. Quantas iterações principais até chegar ao resultado final
- **Como navegar a entrega**: caminho dos arquivos entregues e ordem sugerida de leitura

---

## Critérios de Aceite

A entrega é avaliada contra os critérios abaixo. Todos são obrigatórios.

### PRD (`docs/PRD.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 1
- ☐ Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- ☐ Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- ☐ Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- ☐ Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

### RFC (`docs/RFC.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 2
- ☐ Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
- ☐ Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- ☐ Referencia, com link, pelo menos 2 ADRs do pacote

### FDD (`docs/FDD.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 3
- ☐ Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- ☐ Matriz de erros usa códigos com prefixo `WEBHOOK_`
- ☐ Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- ☐ Seção "Observabilidade" cita métricas, logs e tracing

### ADRs (`docs/adrs/ADR-NNN-*.md`)

- ☐ Pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`
- ☐ Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- ☐ O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
- ☐ Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

### Tracker (`docs/TRACKER.md`)

- ☐ Arquivo existe e segue o formato de tabela definido no requisito 5
- ☐ Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- ☐ Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`
- ☐ Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real

### README (`README.md`)

- ☐ Contém todas as seções obrigatórias listadas no requisito 6
- ☐ Lista pelo menos 1 ferramenta de IA utilizada
- ☐ Mostra pelo menos 2 prompts customizados em blocos de código
- ☐ Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

### Consistência geral

- ☐ Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
- ☐ Nenhum arquivo de código mencionado nos documentos é inexistente no repositório

---

## Estrutura obrigatória do entregável

```
.
├── README.md                              (substituído pelo aluno)
├── TRANSCRICAO.md                         (não alterar)
├── docs/
│   ├── PRD.md                             (preenchido pelo aluno)
│   ├── RFC.md                             (preenchido pelo aluno)
│   ├── FDD.md                             (preenchido pelo aluno)
│   ├── TRACKER.md                         (preenchido pelo aluno)
│   └── adrs/
│       ├── ADR-001-titulo-curto.md
│       ├── ADR-002-titulo-curto.md
│       ├── ADR-003-titulo-curto.md
│       ├── ADR-004-titulo-curto.md
│       ├── ADR-005-titulo-curto.md
│       └── ... (até 8 ADRs)
├── src/                                   (não alterar)
├── prisma/                                (não alterar)
├── tests/                                 (não alterar)
└── ... (demais arquivos do boilerplate)
```

A entrega deve ser feita como repositório público no GitHub, a partir de fork do repositório base do desafio.

## Repositório base

O repositório base do desafio contém a aplicação completa, a transcrição e a estrutura de pastas pra você preencher:

https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

## Ordem de execução sugerida

1. **Fork e setup**: faça o fork do repositório base e clone localmente.
2. **Contextualização com IA**: forneça à IA acesso ao código (via Claude Code, Cursor lendo o repo, ou colando trechos relevantes) e à transcrição. Peça uma exploração inicial para entender estrutura, padrões e o que a feature precisa endereçar.
3. **ADRs primeiro**: identifique e produza as decisões principais antes dos demais documentos. As decisões formam o esqueleto do "como implementar".
4. **RFC**: consolide a proposta técnica em cima das decisões. As alternativas descartadas e as questões em aberto da reunião têm lugar natural aqui. Referencie os ADRs já escritos.
5. **FDD**: com as decisões formalizadas e a proposta consolidada, o desenho técnico se constrói em cima delas. Lembre da seção obrigatória "Integração com o sistema existente".
6. **PRD**: produza o PRD por último entre os grandes documentos. Como ele é mais alto nível, com RFC, FDD e ADRs em mãos vira praticamente uma consolidação.
7. **Tracker**: monte em paralelo com os outros documentos ou no fim, varrendo os documentos prontos.
8. **README do processo**: deixe por último, quando o processo já está completo e você pode documentá-lo com clareza.
9. **Revisão final**: passe pela checklist de critérios de aceite item por item antes do push final.
10. **Itere**: é esperado que o processo demande 3 a 5 ciclos de geração, revisão crítica, ajuste de prompt e nova geração. Se você gerou tudo de primeira sem ajustes, os documentos provavelmente estão genéricos demais.

## Dicas Finais

A qualidade do prompt determina a qualidade do documento. Prompts vagos do tipo "gere um PRD a partir dessa transcrição" produzem documentos vazios e genéricos. Aproveite os prompts disponibilizados pelo professor no curso como base e adapte-os ao contexto deste desafio.

O tracker é seu melhor aliado contra alucinações da IA. Se você não consegue preencher a coluna "Localização" para uma linha do PRD ou do FDD, é sinal de que aquela informação não tem origem identificável e provavelmente foi inventada pela IA. Ajuste ou remova.

Cuidado com o que NÃO entra na documentação. A reunião descarta explicitamente algumas ideias. Se essas coisas aparecerem como requisito nos seus documentos, é sinal de que a IA não está sendo cuidadosa com o que você pediu.

A restrição de não alterar o código da aplicação é absoluta: o código serve de contexto e referência, e o entregável é puramente documental.

Itere bastante. Os primeiros documentos que a IA gerar provavelmente serão superficiais ou redundantes. Volte com correções, peça refinamento de pontos específicos, peça para remover trechos vagos, peça exemplos concretos. O resultado final deve parecer escrito por alguém que pensou no problema com a IA ao lado, não por alguém que copiou e colou da transcrição.

---
---
---

# Sistema de Webhooks de Notificação de Pedidos — Design Docs Gerados por IA | Novo conteúdo solicitado no desafio

Este repositório é minha entrega para o desafio "Da Reunião ao Documento: Design Docs Gerados por IA". O enunciado original completo do desafio está preservado no topo deste mesmo arquivo ("Conteúdo original do desafio") — a partir daqui documento o meu processo de produção, não o enunciado em si.

## Sobre o desafio

O ponto de partida foi um Order Management System já funcional (Node.js, TypeScript, Prisma/MySQL) e a transcrição literal de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`) na qual tech lead, PM, dois engenheiros e uma engenheira de segurança fecharam as decisões de arquitetura para uma nova feature: um sistema de webhooks que notifica clientes B2B sobre mudanças de status de pedido. Nada dessa decisão estava documentado além da própria gravação.

A tarefa não era gerar código, e sim produzir o pacote de documentação técnica (PRD, RFC, FDD, ADRs, Tracker de rastreabilidade) que um time de engenharia usaria para começar a implementar a feature — com a restrição de que toda afirmação nos documentos precisa ser rastreável a um trecho específico da transcrição ou a um arquivo real do código-base, nunca inventada.

## Ferramentas de IA utilizadas

- **Claude Code (Sonnet 5)**, no papel de agente principal de produção: leitura da transcrição e do código, estruturação e escrita de todos os documentos, e orquestração dos subagentes de exploração descritos abaixo.
- **Subagentes `Explore` do próprio Claude Code**, rodados em paralelo, dedicados exclusivamente a extração factual (um mapeando a estrutura de código, outro extraindo decisões/requisitos/timestamps da transcrição) — usados como uma etapa de pesquisa isolada da etapa de escrita, para reduzir o risco de a IA "escrever e inventar ao mesmo tempo".

## Workflow adotado

1. **Exploração paralela e isolada da escrita.** Antes de escrever qualquer documento, disparei dois subagentes em paralelo: um para mapear a estrutura do código (módulos, máquina de estados de pedidos, `changeStatus`, classes de erro, middlewares, logger, schema Prisma) e outro para extrair da transcrição, de forma exaustiva, decisões fechadas, requisitos funcionais/não funcionais, itens descartados, itens adiados, questões em aberto e ganchos com o código — cada um com timestamp `[hh:mm] Nome` ou caminho de arquivo. Separar "pesquisa" de "redação" foi a decisão mais importante do processo: um agente que já está escrevendo prosa tende a preencher lacunas com suposições plausíveis; um agente cujo único objetivo é extrair fatos com citação é mais fácil de auditar linha por linha antes de confiar no resultado.
2. **ADRs primeiro.** Com o material de decisões fechadas em mãos, escrevi os 8 ADRs (6 obrigatórios do enunciado + 2 decisões secundárias com peso arquitetural próprio: snapshot do payload e TLS/limite de tamanho). Cada ADR cita o(s) timestamp(s) da decisão e, quando aplicável, a alternativa real descartada na reunião.
3. **RFC em cima dos ADRs.** O RFC ficou deliberadamente conciso, evitando repetir o detalhe de implementação do FDD: alternativas descartadas (síncrono, Redis Streams, trigger de banco) e questões em aberto (rate limiting, escalabilidade do worker) vieram diretamente da extração da transcrição, e cada decisão proposta linka para o ADR correspondente.
4. **FDD com base no RFC e no mapeamento de código.** A seção "Integração com o sistema existente" foi montada citando os caminhos de arquivo reais levantados na exploração (`order.service.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `logger/index.ts`, `schema.prisma`, `routes/index.ts`, `env.ts`).
5. **PRD por último entre os documentos grandes.** Com RFC, FDD e ADRs prontos, o PRD foi essencialmente uma consolidação em nível de negócio: requisitos funcionais, métricas quantitativas, riscos com probabilidade/impacto/mitigação e a seção de escopo (incluindo os itens explicitamente descartados/adiados na reunião).
6. **Tracker varrendo os documentos prontos.** Percorri PRD, RFC, FDD e cada ADR extraindo um ID por item verificável, com a fonte (`TRANSCRICAO` ou `CODIGO`) e a localização exata — usado também como checklist de auditoria: qualquer item sem localização rastreável seria sinal de conteúdo inventado (não houve nenhum caso).
7. **README por último**, documentando o processo já concluído.

## Prompts customizados

Prompt usado para o subagente de extração da transcrição (a parte mais sensível a alucinação, por isso o pedido explícito de timestamps e trechos literais):

```
Leia o arquivo TRANSCRICAO.md [...] Preciso de uma extração completa e
estruturada, pois será fonte de verdade rastreável para documentos técnicos
(PRD, RFC, FDD, ADRs, Tracker) — cada afirmação nos documentos precisa
apontar para um timestamp [hh:mm] e nome do falante nesta transcrição.

Produza um relatório organizado com:
1. Participantes: nomes e papéis [...]
2. Decisões técnicas fechadas — para cada uma: o que foi decidido,
   timestamp+falante, e se possível o racional/motivo discutido [...]
3. Requisitos funcionais explícitos [...] preciso de pelo menos 8.
4. Requisitos não-funcionais [...]
5. Itens explicitamente descartados durante a reunião [...] com o motivo
   da rejeição
6. Itens adiados para fases futuras [...]
7. Questões deixadas em aberto (não decididas na reunião) [...]
8. Detalhes técnicos secundários [...]
9. Ganchos explícitos com o código existente mencionados na reunião [...]

Seja exaustivo e preciso com os timestamps — isso é crítico para
rastreabilidade. Não resuma demais; cite trechos literais quando relevante.
```

Prompt usado para o subagente de mapeamento de código (focado em não deixar a exploração genérica demais, direcionando exatamente os pontos que os documentos precisariam citar):

```
Investigue e relate (com caminhos de arquivo exatos):
1. Estrutura de pastas/módulos em src/ [...] como cada módulo é organizado
2. O módulo de orders: a máquina de estados [...] e especificamente o
   método changeStatus — como funciona a transação, controle de estoque,
   auditoria de mudança de status
3. Classes de erro existentes (hierarquia, padrão de códigos de erro) [...]
4. O error middleware centralizado [...]
5. O middleware requireRole [...]
6. O logger Pino [...]
7. Schema do Prisma [...] models principais, especialmente Order
8. Padrões de rotas HTTP existentes [...]
9. Configurações de ambiente relevantes

Para cada achado, cite o caminho do arquivo exato e trechos de código
relevantes (breve). Seja específico e completo pois será usado como fonte
de verdade rastreável em documentação técnica — não posso inventar nada.
```

## Iterações e ajustes

- **Separação pesquisa/escrita foi um ajuste deliberado, não a primeira ideia.** A abordagem mais óbvia seria pedir a um único agente para "ler a transcrição e gerar o PRD". Optei por dividir em duas fases (extração factual → redação) justamente para evitar o padrão mais comum de falha nesse tipo de tarefa: um documento fluente e bem escrito, mas com 1-2 requisitos "plausíveis" que na verdade não foram ditos por ninguém na reunião. Com a extração isolada, cada frase de decisão virou uma citação com timestamp que pude conferir contra o texto bruto da transcrição antes de usá-la em qualquer documento.
- **ADRs adicionais além do mínimo do enunciado.** As 6 decisões obrigatórias (outbox, retry/DLQ, HMAC, at-least-once, worker separado, reuso de padrões) cobriam o essencial, mas duas outras decisões da reunião — o snapshot do payload no momento do evento (evitar que um evento reenviado horas depois reflita um status diferente do que motivou a notificação) e a combinação TLS obrigatório + limite de 64KB — tinham alternativas reais discutidas e descartadas ([09:51-09:52] e [09:23], respectivamente) que mereciam registro próprio em vez de ficar diluídas apenas no corpo do FDD. Ajustei o plano original de "5-6 ADRs" para 8 depois de revisar a extração da transcrição e perceber que essas duas decisões tinham peso arquitetural equivalente às demais.
- **Tracker como auditoria, não só entregável.** Ao montar o Tracker por último, usei-o ativamente como checklist de verificação: para cada linha do PRD/RFC/FDD/ADRs, tentei encontrar uma localização exata (timestamp ou caminho de arquivo). Isso não gerou remoções de conteúdo nesta rodada — a extração inicial já havia sido suficientemente disciplinada — mas foi o mecanismo de segurança que teria pego qualquer alucinação antes da entrega final.

## Validação da entrega

### Nota sobre arquivos de código mencionados que ainda não existem

A regra de rastreabilidade do desafio exige que nenhum arquivo de código mencionado na documentação seja inexistente no repositório. Dois caminhos citados nos documentos não existem hoje: `src/worker.ts` e `src/modules/webhooks/webhook.worker.ts` (mencionados em `docs/FDD.md` e em `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` / `docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md`).

Isso não é uma violação da regra: em nenhum dos dois documentos esses caminhos são citados como código já existente — são citados explicitamente como o **entry-point e o módulo novos que a feature precisa criar**, análogos a `src/server.ts` (que esse sim já existe e foi usado como referência de padrão). Todos os demais caminhos de arquivo citados como já existentes no projeto (`src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`, `src/config/env.ts`, `src/routes/index.ts`, `src/app.ts`) foram conferidos e existem de fato no repositório.

### Checklist de critérios de aceite

**PRD (`docs/PRD.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] Identifica no mínimo 8 requisitos funcionais discutidos na reunião (11 identificados)
- [x] Inclui pelo menos 1 objetivo com métrica e meta quantitativa (latência de entrega < 10s)
- [x] Seção "Fora de escopo" lista pelo menos 2 itens descartados/adiados (6 listados)
- [x] Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação (4 listados)

**RFC (`docs/RFC.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] "Alternativas consideradas" lista pelo menos 2 alternativas descartadas com trade-off (3 listadas: síncrono, Redis Streams, trigger de banco)
- [x] "Questões em aberto" lista pelo menos 2 pontos adiados/não decididos (2 listados: rate limiting, escalabilidade do worker)
- [x] Referencia, com link, pelo menos 2 ADRs (referencia os 8)

**FDD (`docs/FDD.md`)**
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] "Contratos públicos" inclui pelo menos 4 endpoints com payload de exemplo e status codes (7 endpoints)
- [x] Matriz de erros usa códigos com prefixo `WEBHOOK_*` (8 códigos)
- [x] "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais (8 caminhos, todos confirmados)
- [x] "Observabilidade" cita métricas, logs e tracing

**ADRs (`docs/adrs/ADR-NNN-*.md`)**
- [x] Pasta contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md` (8 arquivos)
- [x] Cada ADR contém Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- [x] O conjunto cobre pelo menos 5 das 6 decisões principais (cobre as 6)
- [x] Pelo menos 1 ADR referencia arquivos/módulos/classes do código base (ADR-006 referencia 8 caminhos reais)

**Tracker (`docs/TRACKER.md`)**
- [x] Arquivo existe e segue o formato de tabela definido
- [x] Cobertura ampla dos itens identificáveis dos documentos (87 linhas, varredura sistemática de PRD/RFC/FDD/ADRs)
- [x] Pelo menos 70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido `[hh:mm] Nome` (71 de 87 linhas, 81,6%)
- [x] Pelo menos 5 linhas com Fonte = `CODIGO` e caminho de arquivo real (16 linhas)

**README (`README.md`)**
- [x] Contém todas as seções obrigatórias
- [x] Lista pelo menos 1 ferramenta de IA utilizada (Claude Code + subagentes Explore)
- [x] Mostra pelo menos 2 prompts customizados em blocos de código (4 blocos de código, 2 prompts completos)
- [x] Descreve pelo menos 2 iterações/ajustes concretos (separação pesquisa/escrita; expansão de 6 para 8 ADRs; Tracker como auditoria)

**Consistência geral**
- [x] Nenhum requisito, decisão ou restrição registrada contradiz a transcrição ou o código
- [x] Nenhum arquivo de código mencionado como já existente é inexistente no repositório (ver nota acima sobre `src/worker.ts` e `webhook.worker.ts`, que são propostas de arquivos novos, não afirmações sobre código já existente)

## Como navegar a entrega

Ordem sugerida de leitura, do mais estratégico ao mais tático:

1. [`docs/PRD.md`](docs/PRD.md) — por quê e o quê: problema de negócio, requisitos, escopo, riscos.
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica de alto nível, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 8 decisões de arquitetura individuais (ADR-001 a ADR-008), cada uma com contexto, decisão, alternativas e consequências.
4. [`docs/FDD.md`](docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — rastreabilidade de cada item dos documentos acima até a transcrição ou o código-fonte.
6. `TRANSCRICAO.md` — fonte primária, útil para conferir qualquer citação `[hh:mm] Nome` referenciada nos documentos.
7. `src/`, `prisma/` — código-base existente, referenciado (mas não alterado) pela documentação, especialmente por `docs/FDD.md`.
