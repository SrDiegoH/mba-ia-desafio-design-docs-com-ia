# ADR-008: TLS obrigatório e limite rígido de tamanho de payload

## Status

Aceito

## Contexto

Além da autenticação por HMAC (ver [[ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint]]), a revisão de segurança da reunião (conduzida por Sofia) levantou duas restrições adicionais sobre o transporte das notificações: o protocolo aceito para as URLs de destino cadastradas pelos clientes, e o comportamento do sistema diante de payloads potencialmente grandes.

## Decisão

**TLS obrigatório**: toda URL de webhook cadastrada deve usar `https`, validado via schema Zod na criação/edição da configuração — [09:23, Sofia]. URLs `http` são rejeitadas na validação de entrada, antes de qualquer tentativa de entrega.

**Limite de payload de 64KB**: se o payload de um evento ultrapassar 64KB, o sistema **retorna erro explicitamente**, em vez de truncar o conteúdo.

> "Eu sou a favor de erra. Se chegou nesse tamanho, tem algo errado." — [09:23, Sofia]

Discussão confirmada com Diego e Larissa concordando com o limite e o comportamento de falha explícita — [09:23-09:24, Sofia/Diego/Larissa].

## Alternativas Consideradas

**Permitir `http` para clientes que não suportem TLS.** Simplificaria a integração para clientes com infraestrutura legada, mas exporia o payload (mesmo assinado por HMAC) e os headers de autenticação a interceptação em trânsito, e foi descartada sem contestação durante a reunião — segurança de transporte tratada como não-negociável por Sofia — [09:23, Sofia].

**Truncar o payload quando exceder o limite de 64KB.** Manteria a entrega "funcionando" mesmo com payloads grandes, mas mascararia um problema real (payload inflando por bug ou mudança de escopo não avaliada) e entregaria dados incompletos/corrompidos ao cliente sem sinalização clara. Descartada em favor de falhar de forma explícita e visível: "Eu sou a favor de erra." — [09:23, Sofia]

## Consequências

**Positivas**
- Elimina a superfície de ataque de transporte em texto claro (man-in-the-middle) para todas as notificações enviadas pelo OMS.
- Falha explícita em payloads grandes evita entregar dados truncados/inconsistentes ao cliente e sinaliza cedo um problema de modelagem (ex.: um novo campo inflando o payload sem revisão).
- Validação de URL na camada de schema Zod (na criação/edição do webhook) segue o mesmo padrão de validação já usado nos demais módulos do projeto.

**Negativas**
- Exclui clientes que só conseguem expor endpoints HTTP simples (sem TLS), o que pode ser uma barreira de adoção para integrações legadas — trade-off aceito conscientemente em favor de segurança.
- Um payload que ultrapasse 64KB resulta em evento não entregue (erro), exigindo tratamento explícito (ver matriz de erros `WEBHOOK_*` no FDD) e, potencialmente, intervenção humana para diagnosticar por que o payload cresceu além do esperado.
