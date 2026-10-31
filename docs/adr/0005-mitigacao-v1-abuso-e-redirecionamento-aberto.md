# ADR 0005 — Mitigação v1 de abuso de criação e de redirecionamento aberto

> Status: aceito (TRD-2 e TRD-3 confirmados pelo usuário em 2026-10-31, sem alteração em relação
> à proposta original)

## Contexto

O PRD (seção 9, indicador "Segurança") sinaliza dois riscos sem decidir a solução, deixando a
decisão explicitamente para o `architect`: (a) redirecionamento aberto — qualquer código
encurtado pode levar a qualquer destino, incluindo phishing/malware; (b) uso anônimo combinado com
alto tráfego pode permitir criação de links em massa (spam). Nenhum dos dois é um critério de
aceite do PRD (não há história de usuário sobre bloquear links maliciosos ou sobre limite de
taxa) — são indicadores técnicos a endereçar no desenho, não funcionalidades visíveis ao usuário
final.

## Decisão

**v1 implementa mitigação básica, não abrangente**, para os dois riscos:

- **Redirecionamento aberto**: a URL original é validada quanto ao **esquema** apenas — somente
  `http://` e `https://` são aceitos; qualquer outro esquema (`javascript:`, `data:`, `file:`,
  etc.) é rejeitado como "URL inválida" (mesmo caminho de erro do cenário "URL malformada" de
  US-1). **Não há verificação de reputação/moderação de conteúdo do destino** (ex.: consulta a uma
  API de reputação como Google Safe Browsing) nesta versão — o sistema aceita qualquer destino
  `http(s)` sintaticamente válido, mesmo que leve a um site malicioso.
- **Abuso de criação em massa**: limite de taxa (rate limiting) por endereço IP do requisitante,
  janela fixa (contador Redis com TTL), limite configurável via variável de ambiente
  (`RATE_LIMIT_CREATE_MAX_PER_MINUTE`, **confirmado pelo usuário em 20** requisições/minuto por
  IP — TRD-3). Requisições acima do limite recebem erro `RATE_LIMITED` (HTTP 429).

## Alternativas consideradas

- **Integração com API de reputação de URL (ex.: Google Safe Browsing) antes de aceitar a
  criação**: mitigaria o risco de phishing/malware de forma muito mais efetiva, mas introduz uma
  dependência externa paga/com cota, latência adicional no caminho de criação, e complexidade de
  resiliência (o que fazer se a API de reputação estiver fora do ar?) desproporcional ao escopo
  desta primeira spec. Fica registrada como pendência explícita (TRD, seção 15) para o usuário
  decidir se entra numa fatia/spec futura.
- **CAPTCHA ou prova de trabalho na criação**: mitigaria abuso de forma mais robusta que rate
  limit por IP (que não impede um atacante distribuído com múltiplos IPs), mas adiciona atrito à
  experiência de uso anônimo que o PRD define como objetivo central ("poucos segundos, sem passos
  extras", seção 4). Rejeitada para v1 por conflitar com um objetivo explícito do PRD.
- **Bloqueio de lista de domínios conhecidos maliciosos mantida localmente**: exigiria manter e
  atualizar uma lista própria, sem fonte de dados confiável sem integração externa — equivalente
  em complexidade a adotar uma API de reputação, mas com qualidade inferior. Rejeitada.

## Consequências

- O sistema aceita, por design nesta v1, links que podem apontar a destinos maliciosos — risco
  aceito conscientemente e repassado ao `security-engineer` (etapa de revisão de segurança desta
  spec) para avaliação mais profunda, e ao usuário como pendência de produto (decidir se uma
  integração de reputação é prioridade de uma spec futura).
- Rate limiting por IP não impede abuso distribuído (múltiplos IPs); é uma mitigação de custo
  baixo contra o caso mais comum (um único cliente abusando em volume), não uma defesa completa.
- O valor do limite de taxa (`RATE_LIMIT_CREATE_MAX_PER_MINUTE=20`) e a ausência de verificação de
  reputação de destino estão **confirmados pelo usuário** (TRD-3 e TRD-2, respectivamente) — sem
  pendência em aberto para esta ADR.
