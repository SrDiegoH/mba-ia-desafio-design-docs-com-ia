# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

Webhooks são requisições HTTP que o OMS envia para URLs controladas pelos clientes. É necessário que o cliente consiga verificar que o payload recebido realmente veio do OMS (autenticidade) e não foi alterado em trânsito (integridade), sem depender apenas de TLS. Também é preciso decidir o escopo de compartilhamento de segredos: uma única chave para toda a plataforma ou uma chave por integração.

## Decisão

Assinar cada payload de webhook com **HMAC-SHA256**, calculado sobre o corpo (body) do request, e enviado no header `X-Signature`.

> "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso." — [09:20, Sofia]

Cada **endpoint de webhook cadastrado por cliente tem sua própria secret**, gerada pelo sistema no momento da criação — não existe uma secret global compartilhada entre integrações.

> "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo." — [09:21, Sofia]

O sistema também deve suportar **rotação de secret com grace period de 24 horas**, durante o qual tanto a secret antiga quanto a nova são aceitas como válidas — [09:21, Sofia]. O racional para suportar rotação vem de um incidente real já observado pelo time: "A gente já teve cliente que vazou secret em log de aplicação dele uma vez." — [09:22, Diego]

> "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24 horas." — [09:22, Sofia]

Complementarmente, foi definido que as URLs de destino devem ser obrigatoriamente HTTPS (validado via schema Zod) — [09:23, Sofia] — e que o request de entrega deve incluir também o header `X-Timestamp`, permitindo ao cliente detectar ataques de replay — [09:44-09:45, Sofia].

## Alternativas Consideradas

**Secret única global por plataforma (compartilhada entre todos os clientes/endpoints).** Mais simples de gerar e gerenciar, mas concentra o risco: o vazamento de uma única secret comprometeria a autenticidade de notificações para todos os clientes simultaneamente. Descartada pelo argumento direto de Sofia: "Senão se vaza uma, vaza tudo." — [09:21, Sofia]

**Assinatura sem suporte a rotação (secret fixa, sem grace period).** Mais simples de implementar, mas torna qualquer vazamento de secret um evento crítico sem caminho de remediação sem downtime — o cliente precisaria trocar a secret e resincronizar exatamente no mesmo instante que o OMS, sem margem de erro. Descartada em favor do grace period de 24h após o time relatar um incidente real de vazamento — [09:21-09:22, Sofia/Diego].

## Consequências

**Positivas**
- Autenticidade e integridade do payload verificáveis pelo cliente sem depender de infraestrutura adicional (HMAC é padrão amplamente suportado por bibliotecas HTTP client de mercado).
- Isolamento de blast radius: vazamento de uma secret afeta apenas um endpoint/cliente, não a base inteira.
- Rotação com grace period permite resposta a incidentes de vazamento sem downtime de integração para o cliente.

**Negativas**
- Aumenta a complexidade de modelagem de dados: é preciso armazenar (de forma seura) potencialmente duas secrets ativas simultaneamente por endpoint durante o período de rotação, e um mecanismo de expiração da secret antiga após as 24h.
- Requer geração e armazenamento seguro de segredos por endpoint (fora do escopo desta decisão, mas relevante para segurança operacional — ex. hashing/encriptação em repouso).
- HMAC exige que o cliente implemente a verificação corretamente do seu lado; não há como o OMS garantir que o cliente de fato valida a assinatura recebida.
