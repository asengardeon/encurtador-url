# ADR 0004 — Geração de código curto aleatório com detecção de colisão via SETNX

> Status: aceito (confirmado pelo usuário em 2026-10-31 junto da stack — ver `docs/STACK.md`;
> mecanismo de geração atualizado de `crypto.randomInt` do Node para `secrets` do Python após a
> troca de backend, ADR 0002)

## Contexto

O PRD (seção 5, "Fora de escopo") decide explicitamente que não há deduplicação: encurtar a mesma
URL duas vezes gera um código novo a cada vez (US-1, cenário "Encurtar a mesma URL mais de uma
vez"). Isso significa que o sistema não pode derivar o código a partir de um hash determinístico
da URL (ex.: hash truncado) sem um mecanismo de desambiguação, porque a mesma URL precisa poder
gerar códigos diferentes em chamadas distintas. O PRD também sinaliza (seção 9) que o
`architect` precisa decidir a "taxa de geração de códigos sem colisão" sob alto tráfego.

## Decisão

Código curto gerado como string aleatória base62 (dígitos + letras maiúsculas/minúsculas) de 7
caracteres, usando o gerador criptograficamente seguro da stdlib do Python (`secrets.choice`
sobre o alfabeto base62) para cada caractere. A escrita no Redis usa `SET shortlink:<código> <url>
NX` (set-if-not-exists) —
atômico por construção, porque o Redis executa comandos únicos de forma serializada
(`[não medido]`, a confirmar por quem implementar com teste de integração real contra a versão
exata do Redis usada, não só por documentação). Se `NX` falhar (código já existe), o caso de uso
`CreateShortLink` gera um novo candidato e tenta novamente, até um limite de tentativas (5,
configurável) antes de retornar um erro de "espaço de códigos esgotado" — cenário estatisticamente
improvável no espaço de 62^7 combinações, mas tratado explicitamente em vez de assumido impossível.

## Alternativas consideradas

- **Contador incremental (auto-increment) codificado em base62**: gera códigos sequenciais,
  previsíveis (um atacante pode enumerar todos os links já criados incrementando o contador) — o
  próprio PRD sinaliza redirecionamento aberto como risco de segurança; previsibilidade do código
  piora esse risco ao facilitar descoberta de links de terceiros. Rejeitada.
- **Hash truncado da URL + timestamp/nonce para evitar colisão determinística**: resolveria a
  exigência de "código novo a cada encurtamento", mas é mais complexo que gerar um valor aleatório
  direto sem ganho real, já que não há necessidade de determinismo (sem deduplicação, não importa
  se o código "lembra" a URL).
- **Reservar um bloco de códigos por instância do backend** (para reduzir contenção de escrita
  concorrente): otimização prematura — não há evidência ainda de que a taxa de colisão real
  justifique essa complexidade; revisar se a métrica de tentativas de retry em produção mostrar
  taxa de colisão relevante.

## Consequências

- Nenhuma coordenação externa (lock distribuído) é necessária além do próprio `SETNX` atômico do
  Redis.
- O código gerado não é previsível, mitigando parcialmente o risco de enumeração de links de
  terceiros.
- Sob volume alto o suficiente para a taxa de colisão de 7 caracteres base62 deixar de ser
  desprezível, o limite de retry pode começar a ser atingido com frequência perceptível — métrica
  de observabilidade (TRD, seção 10) cobre isso para decisão futura (aumentar o tamanho do código,
  por exemplo).
- Esta decisão está **confirmada pelo usuário** (`docs/STACK.md`, nota de status) — a troca de
  backend para Python (ADR 0002) só muda a API de geração de aleatoriedade usada
  (`secrets.choice` em vez de `crypto.randomInt`), não o algoritmo nem o mecanismo de detecção de
  colisão.
