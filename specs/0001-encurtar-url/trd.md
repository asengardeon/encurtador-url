# TRD — Encurtar URL

> Status: aprovado (TRD-5, não bloqueante, permanece como pendência — ver "Pendência restante" abaixo)
> Autor: agente `architect`
> PRD relacionado: `specs/0001-encurtar-url/prd.md`
> ADRs relacionados: `docs/adr/0002-stack-tecnologica-encurtador-url.md`,
> `docs/adr/0003-redis-como-armazenamento-primario.md`,
> `docs/adr/0004-geracao-codigo-curto-sem-colisao.md`,
> `docs/adr/0005-mitigacao-v1-abuso-e-redirecionamento-aberto.md`

## Pendência restante (ler antes do resto)

As duas pré-condições que bloqueavam este TRD na rodada anterior foram resolvidas:

1. **Remote GitHub**: configurado — `https://github.com/asengardeon/encurtador-url` (privado),
   `origin`. As 13 issues da seção 13 foram criadas (milestone `0001-encurtar-url`), com a coluna
   "Issue GitHub" preenchida.
2. **Stack tecnológica (seção 2)**: confirmada pelo usuário em 2026-10-31, com uma correção em
   relação à proposta original — o backend passa a ser **Python + FastAPI assíncrono** em vez de
   Node.js/Fastify (frontend e Redis permanecem como propostos). Detalhe em
   `docs/adr/0002-stack-tecnologica-encurtador-url.md`.

Resta só **uma pendência não bloqueante** (TRD-5, seção 15: domínio real de produção para
`SHORT_URL_BASE_DOMAIN`) e a **aprovação explícita do usuário** (seção 17) — decisão que não é do
`architect`, só repassada a ele.

## 1. Contexto

Um encurtador de URL público e anônimo: o usuário cola uma URL longa, recebe um código curto, e
qualquer pessoa que acessar esse código é redirecionada para a URL original — sem contas, sem
histórico, sem deduplicação, com uso público de alto tráfego esperado. A abordagem técnica:
backend stateless e assíncrono (Python/FastAPI sobre Uvicorn) com Redis como armazenamento primário (não cache) do
mapeamento código→URL, geração de código aleatória com detecção de colisão atômica, e um frontend
de tela única que fala com o backend por um contrato REST simples — permitindo
`backend-developer` e `frontend-developer` trabalharem em paralelo desde a primeira fatia.

## 2. Stack Tecnológica

- **Linguagem/runtime**: Python 3.12+ no backend; TypeScript sobre Node.js 22 LTS só como
  toolchain de build do frontend (o runtime de produção do frontend é o navegador).
- **Framework principal**: FastAPI sobre Uvicorn (ASGI), **assíncrono de ponta a ponta** —
  rotas `async def`, cliente Redis assíncrono — no backend; nenhum framework de componentização
  no frontend (TypeScript vanilla + Vite).
- **Persistência**: Redis 7.x, armazenamento primário (não cache) do mapeamento código→URL, com
  persistência AOF habilitada. Cliente Python: `redis.asyncio` (pacote `redis-py`).
- **Gerenciador de pacotes**: `pip` + `pyproject.toml` no backend; `npm` no frontend (projeto
  independente em `frontend/`, sem workspace compartilhado).
- **Validação de variáveis de ambiente**: `pydantic-settings` (seção 9).
- **Proveniência desta decisão**: proposta inicial do `architect` (sem acesso a
  `AskUserQuestion` nesta invocação) **corrigida pelo usuário em 2026-10-31** — o backend passa
  de Node.js/Fastify para Python/FastAPI assíncrono (motivo do usuário: encaixa melhor com o
  indicador de alto tráfego do PRD e com um cliente Redis assíncrono, sem bloquear o event loop
  no caminho de redirecionamento). Frontend e Redis confirmados sem alteração. Detalhe completo e
  alternativas consideradas em `docs/adr/0002-stack-tecnologica-encurtador-url.md`.

Esta é a primeira vez que a stack é decidida neste projeto. `docs/STACK.md` foi atualizado com
este mesmo conteúdo (incluindo comandos de execução de teste e o mecanismo de isolamento do Redis
de teste entre execuções concorrentes) — features seguintes reaproveitam a partir de lá.

## 3. Visão de arquitetura

```
[HTTP: POST /api/links]  ──▶ CreateShortLink (caso de uso) ──▶ ShortLinkWriterPort     ◀── RedisShortLinkRepository
   (CreateShortLinkHttpController)   │                    ──▶ ShortCodeGeneratorPort  ◀── Base62RandomShortCodeGenerator
                                      │                    ──▶ RateLimiterPort         ◀── RedisFixedWindowRateLimiter
                                      ▼
                             domain: ShortLink, ShortCode, OriginalUrl

[HTTP: GET /:code]        ──▶ RedirectToOriginalUrl (caso de uso) ──▶ ShortLinkReaderPort ◀── RedisShortLinkRepository
   (RedirectHttpController)          │
                                      ▼
                             domain: ShortLink, ShortCode

[frontend/src/components/CardEncurtador] ──▶ [frontend/src/services/LinkService] ──(contrato seção 8)──▶ backend acima
```

`RedisShortLinkRepository` implementa tanto `ShortLinkWriterPort` quanto `ShortLinkReaderPort`
(duas interfaces pequenas, uma classe de adapter — ver seção 5 sobre por que são dois ports e não
um "repositório genérico"). O frontend nunca fala com Redis nem conhece a existência dele; só
conhece o contrato HTTP da seção 8, através de `LinkService` (o equivalente frontend de um adapter
de saída, `docs/ARCHITECTURE.md`).

## 4. Modelo de domínio

- **`OriginalUrl`** (value object): envolve uma string, validando no construtor que é uma URL
  absoluta bem formada com esquema `http` ou `https` (qualquer outro esquema, ou string que não
  parseia como URL, lança `InvalidUrlError`; string vazia lança `EmptyUrlError` — dois erros
  distintos porque US-1 tem dois cenários de erro distintos: campo vazio vs. URL malformada).
  Imutável após criação.
- **`ShortCode`** (value object): envolve uma string de exatamente 7 caracteres do alfabeto base62
  (`[0-9A-Za-z]`). Valida formato no construtor — usado tanto para códigos recém-gerados quanto
  para o parâmetro de rota recebido em `GET /:code` (um código com formato inválido já é
  "não encontrado" antes de qualquer consulta ao Redis).
- **`ShortLink`** (entidade): par imutável `(code: ShortCode, originalUrl: OriginalUrl)`.
  Invariante: uma vez criado, nem `code` nem `originalUrl` mudam — não há operação de edição ou
  expiração nesta spec (PRD, seção 5). Igualdade por `code` (é a chave de identidade).

Nenhuma dessas três classes importa Redis, FastAPI, ou qualquer outra infraestrutura — são Python
puro (dataclasses imutáveis, `docs/ARCHITECTURE.md`, regra da dependência).

## 5. Ports (contratos)

### `ShortLinkWriterPort`

- Responsabilidade única: persistir um novo `ShortLink`, detectando colisão de código de forma
  atômica.
- Método: `async def try_save(link: ShortLink) -> Literal['saved', 'code_already_exists']` —
  nunca lança por colisão (colisão é um retorno esperado, não uma exceção); lança
  `StorageUnavailableError` se o armazenamento estiver inacessível (timeout/conexão recusada).
- Quem implementa: `RedisShortLinkRepository` (`SET shortlink:<code> <url> NX`).
- Quem consome: caso de uso `CreateShortLink`.

### `ShortLinkReaderPort`

- Responsabilidade única: buscar o `ShortLink` por código.
- Método: `async def find_by_code(code: ShortCode) -> ShortLink | None` — `None` quando não existe
  (não é um erro; é o cenário "link não encontrado" de US-3). Lança `StorageUnavailableError` se o
  armazenamento estiver inacessível.
- Quem implementa: `RedisShortLinkRepository` (`GET shortlink:<code>`).
- Quem consome: caso de uso `RedirectToOriginalUrl`.
- **Por que não é o mesmo port que `ShortLinkWriterPort`** (Interface Segregation,
  `docs/ARCHITECTURE.md`): `CreateShortLink` só precisa escrever; `RedirectToOriginalUrl` só
  precisa ler. Um "repositório genérico" com os dois métodos obrigaria cada caso de uso a depender
  de uma capacidade que não usa — mesmo com uma única classe de adapter implementando as duas
  interfaces, o que importa para o teste do caso de uso é o port que ele declara, não a classe
  concreta por trás.

### `ShortCodeGeneratorPort`

- Responsabilidade única: produzir um candidato de código curto aleatório.
- Método: `def generate() -> ShortCode` (síncrono — geração de código é CPU-bound, não há I/O a
  aguardar).
- Quem implementa: `Base62RandomShortCodeGenerator` (`secrets.choice` por caractere,
  `docs/adr/0004-geracao-codigo-curto-sem-colisao.md`).
- Quem consome: caso de uso `CreateShortLink`.
- Por que é um port (e não uma função pura chamada direto): geração de código é não determinística
  por natureza (é aleatória de propósito) — tratá-la como port permite que o teste unitário de
  `CreateShortLink` injete um gerador determinístico (fake que devolve uma sequência fixa de
  códigos) para testar o fluxo de retry em colisão sem depender de aleatoriedade real
  (testabilidade em primeiro lugar, `.claude/agents/architect.md`).

### `RateLimiterPort`

- Responsabilidade única: decidir se uma requisição de criação, identificada pelo IP do
  requisitante, pode prosseguir.
- Método: `async def check_and_consume(client_ip: str) -> Literal['allowed', 'limited']`.
- Quem implementa: `RedisFixedWindowRateLimiter` (`INCR` + `EXPIRE` por IP, janela fixa,
  `docs/adr/0005-mitigacao-v1-abuso-e-redirecionamento-aberto.md`).
- Quem consome: caso de uso `CreateShortLink`.

Nenhum destes ports resolve identidade/escopo por um parâmetro ambíguo (tenant/workspace) — esta
feature não tem conceito de tenant, conta, nem contexto operacional distinto da identidade do
requisitante (uso totalmente anônimo), então o item "identidade vs. escopo" do processo do
`architect` (passo 4a) não se aplica aqui.

## 6. Casos de uso

| Critério de aceite (PRD) | Caso de uso | Ports usados |
|---------------------------|---------------|----------------|
| US-1, cenário "Encurtar uma URL válida" | `CreateShortLink` | `RateLimiterPort`, `ShortCodeGeneratorPort`, `ShortLinkWriterPort` |
| US-1, cenário "Tentar encurtar com o campo vazio" | `CreateShortLink` (lança `EmptyUrlError` antes de consultar qualquer port) | nenhum (validação de domínio, `OriginalUrl`) |
| US-1, cenário "Tentar encurtar uma URL malformada" | `CreateShortLink` (lança `InvalidUrlError` antes de consultar qualquer port) | nenhum (validação de domínio, `OriginalUrl`) |
| US-1, cenário "Feedback de carregamento durante o encurtamento" | não aplicável a caso de uso de backend — é estado de UI do frontend enquanto a chamada HTTP está em voo (seção 11, plano de testes de frontend) | — |
| US-1, cenário "Encurtar a mesma URL mais de uma vez" | `CreateShortLink` (nenhuma verificação de URL já existente — gera e persiste um `ShortCode` novo a cada chamada, por design; ver seção 4) | `ShortCodeGeneratorPort`, `ShortLinkWriterPort` |
| US-2 (copiar com feedback de sucesso) | não aplicável a caso de uso de backend — Clipboard API é inteiramente client-side (seção 11, plano de testes de frontend) | — |
| US-3, cenário "Redirecionamento bem-sucedido" | `RedirectToOriginalUrl` | `ShortLinkReaderPort` |
| US-3, cenário "Acesso a uma URL encurtada que não existe" | `RedirectToOriginalUrl` (retorna `None`, adapter de entrada traduz para página "não encontrado") | `ShortLinkReaderPort` |

**`CreateShortLink`** — entrada: `{ original_url_raw: str, client_ip: str }`. Fluxo: (1)
constrói `OriginalUrl` (lança `EmptyUrlError`/`InvalidUrlError` se inválida); (2)
`RateLimiterPort.check_and_consume(client_ip)` — se `'limited'`, lança `RateLimitedError`; (3)
laço de até 5 tentativas: gera `ShortCode` via `ShortCodeGeneratorPort`, monta `ShortLink`, tenta
`ShortLinkWriterPort.try_save`; se `'code_already_exists'`, tenta de novo; se esgotar tentativas,
lança `CollisionExhaustedError`; (4) retorna o `ShortLink` salvo. DTO de retorno:
`{ code: str, original_url: str }` — **não inclui** a URL curta completa (`shortUrl`); essa
composição é resolvida na borda do adapter de entrada HTTP (seção 9), porque o domínio base
(`short.ly`, ou o que for configurado) é uma configuração de infraestrutura, não uma regra de
negócio.

**`RedirectToOriginalUrl`** — entrada: `{ code_raw: str }`. Fluxo: (1) tenta construir
`ShortCode` a partir de `code_raw` — se o formato já for inválido, retorna "não encontrado" sem
consultar o port (um código malformado nunca existiu); (2) `ShortLinkReaderPort.find_by_code` —
se `None`, retorna "não encontrado"; (3) retorna `{ original_url: str }`.

## 7. Adapters

### Entrada

- **`CreateShortLinkHttpController`** (`POST /api/links`, rota `async def` do FastAPI): lê
  `{ url }` do corpo da requisição (modelo Pydantic) e o IP do requisitante (`request.client.host`
  do Starlette/FastAPI — assume que, se o app roda atrás de um proxy reverso, o Uvicorn está
  configurado com `--proxy-headers` e `--forwarded-allow-ips` corretos para que
  `request.client.host` reflita o IP real do cliente e não o do proxy; decisão de infraestrutura
  sinalizada ao `sre` na seção 10), chama `CreateShortLink`, e traduz o resultado/exceção para a
  resposta HTTP do contrato da seção 8. Implementado pelo `backend-developer`.
- **`RedirectHttpController`** (`GET /:code`): chama `RedirectToOriginalUrl` com o parâmetro de
  rota; em sucesso, responde `302 Found` com header `Location: <originalUrl>`; em "não
  encontrado", responde `404` com uma página HTML mínima e estática (server-rendered, não é uma
  rota da SPA — quem acessa este endpoint é o navegador de quem recebeu o link, não o frontend via
  `fetch`). Implementado pelo `backend-developer`.

### Saída

- **`RedisShortLinkRepository`**: implementa `ShortLinkWriterPort` e `ShortLinkReaderPort`.
  Cliente `redis.asyncio` (pacote `redis-py`). Chave Redis: `shortlink:<code>` → valor: a string
  da URL original (ver seção 9 para o porquê de não guardar mais que isso). Implementado pelo
  `backend-developer`.
- **`Base62RandomShortCodeGenerator`**: implementa `ShortCodeGeneratorPort`. Sem dependência
  externa (usa `secrets` da stdlib do Python). Implementado pelo `backend-developer`.
- **`RedisFixedWindowRateLimiter`**: implementa `RateLimiterPort`. Chave Redis:
  `ratelimit:create:<ip>`, `INCR` + `EXPIRE` de 60s, limite configurável via
  `RATE_LIMIT_CREATE_MAX_PER_MINUTE` (variável de ambiente nova — ver seção 13, critério de
  aceite de `.env.example`/validador de ambiente). Implementado pelo `backend-developer`.

Nenhum destes adapters de saída fala com um serviço gerenciado de nuvem (S3/DynamoDB/etc.) — a
tabela "Simulação de nuvem local" de `docs/STACK.md` permanece vazia por esse motivo.

## 8. Contrato Frontend↔Backend (API)

- **Aplicável?** Sim — feature full-stack.

### `POST /api/links`

- **Request**: `{ "url": string }`
- **Response (sucesso, `201`)**:
  ```json
  { "shortCode": "aB3xZ9k", "shortUrl": "https://short.ly/aB3xZ9k", "originalUrl": "https://exemplo.com/produtos/categoria/pagina-muito-longa" }
  ```
  `shortUrl` é montado pelo adapter HTTP concatenando `SHORT_URL_BASE_DOMAIN` (variável de
  ambiente, seção 13) com `shortCode` — não vem do caso de uso (seção 6). Os nomes de campo no
  JSON permanecem em `camelCase` (convenção comum de API web) independentemente de o backend ser
  Python — o modelo Pydantic do `CreateShortLinkHttpController` declara `alias` camelCase para
  cada campo (`populate_by_name`/`alias_generator`), sem expor `snake_case` no contrato.
- **Response (erro)**: corpo padrão `{ "error": { "code": string, "message": string } }`, onde
  `code` é um dos quatro valores abaixo e `message` é texto em português pronto para exibição ao
  usuário.

| HTTP status | `error.code` | Quando |
|---------------|-----------------|----------|
| 400 | `EMPTY_URL` | Campo de URL vazio (US-1, cenário "campo vazio") |
| 400 | `INVALID_URL` | URL malformada ou com esquema não permitido (US-1, cenário "URL malformada") |
| 429 | `RATE_LIMITED` | Limite de criação por IP excedido (`docs/adr/0005-...md`) |
| 500 | `INTERNAL_ERROR` | `CollisionExhaustedError`, falha de armazenamento, ou qualquer erro não mapeado |

### `GET /:code`

- Não é uma chamada da SPA via `fetch` — é a navegação do navegador de quem recebeu o link
  encurtado (US-3). Sem corpo de request.
- **Resposta (sucesso)**: `302 Found`, header `Location: <originalUrl>`, sem corpo relevante.
- **Resposta (não encontrado)**: `404`, página HTML estática mínima ("este link não existe ou foi
  removido"), server-rendered pelo próprio backend (sem dependência do bundle da SPA).

### Autenticação/autorização

Nenhuma — uso totalmente anônimo (PRD, seção 3).

### ADR relacionado

Nenhum — este contrato é específico desta feature, não uma convenção reutilizável por todo o app
ainda (se um app futuro tiver múltiplos endpoints, o formato de erro `{ error: { code, message } }`
é candidato a ser promovido a ADR então).

## 9. Modelo de dados / contratos externos

### Redis — chaves de dados

| Chave | Valor | TTL | Escrito por | Lido por |
|--------|--------|-------|----------------|-----------|
| `shortlink:<code>` | string da URL original | nenhum (permanente, PRD seção 5) | `RedisShortLinkRepository.try_save` | `RedisShortLinkRepository.find_by_code` |
| `ratelimit:create:<ip>` | contador inteiro | 60s (janela fixa) | `RedisFixedWindowRateLimiter.check_and_consume` | mesmo método |

O valor de `shortlink:<code>` é só a string da URL — nenhum metadado (data de criação, IP de
quem criou) é persistido junto, porque o PRD exclui histórico/métricas visíveis (seção 5) e esta
spec não precisa consultar esse metadado depois. Observabilidade de criação (seção 10) é coberta
por log estruturado no momento da escrita, não por campo persistido — se uma spec futura
(`0002-expirar-link`, por exemplo) precisar de um campo adicional (ex.: timestamp de expiração), o
formato do valor passa a ser JSON em vez de string simples; essa mudança é decidida na spec que
precisar dela, não aqui.

### Variáveis de ambiente novas

| Variável | Uso | Validada em |
|-----------|------|----------------|
| `REDIS_URL` | string de conexão do `redis.asyncio` | validador de ambiente do backend, `pydantic-settings` (seção 13, critério de aceite) |
| `SHORT_URL_BASE_DOMAIN` | domínio público usado para montar `shortUrl` na resposta de `POST /api/links` | idem |
| `RATE_LIMIT_CREATE_MAX_PER_MINUTE` | limite do `RedisFixedWindowRateLimiter` (padrão sugerido: 20 — pendência TRD-3, seção 15) | idem |

Todo campo do contrato da seção 8 tem fonte explícita: `shortCode`/`originalUrl` vêm do DTO de
retorno de `CreateShortLink`/`RedirectToOriginalUrl` (seção 6); `shortUrl` é resolvido na borda do
`CreateShortLinkHttpController` a partir de `SHORT_URL_BASE_DOMAIN` (declarado explicitamente
acima, não uma lacuna implícita).

## 10. Pilares de engenharia de software

- **Performance**: o caminho de redirecionamento (`GET /:code`) é o mais exercitado sob o alto
  tráfego que o PRD sinaliza (seção 9) — `GET` no Redis é O(1), então a latência dominante deve
  ser rede/overhead do FastAPI/Uvicorn, não o armazenamento. Backend **assíncrono de ponta a
  ponta** (FastAPI + `redis.asyncio`, ADR 0002): enquanto uma requisição espera a resposta do
  Redis, o event loop do Uvicorn atende outras requisições concorrentes, em vez de bloquear uma
  thread por requisição — motivo pelo qual o usuário corrigiu a stack original para esta
  combinação. O caminho de criação (`POST /api/links`) tolera latência um pouco maior (laço de
  retry em colisão), mas cada tentativa individual (`SET NX`) também é O(1).
- **Escalabilidade**: o backend não guarda nenhum estado em memória do processo — todo o
  mapeamento e todo o contador de rate limit vivem no Redis. Múltiplas instâncias do backend
  podem rodar atrás de um balanceador de carga compartilhando a mesma instância Redis sem
  divergência de comportamento (inclusive o rate limit, que precisa ser compartilhado entre
  instâncias para não virar "20 requisições por instância" em vez de "20 por IP").
- **Resiliência**: a única dependência externa é o Redis. Decisão: timeout explícito de conexão e
  de comando no cliente `redis.asyncio` (parâmetros `socket_connect_timeout`/`socket_timeout`,
  valores a definir pelo `backend-developer` dentro de uma ordem de grandeza de poucos segundos)
  — nunca esperar
  indefinidamente. Sem retry automático de comando nem circuit breaker nesta v1: não há fallback
  possível (não existe segunda fonte de dados), então uma falha do Redis se traduz direto em
  `503`/`INTERNAL_ERROR` ao requisitante, de forma rápida e explícita, em vez de mascarada por
  tentativas repetidas. Risco de ponto único de falha registrado na seção 12 e endereçado como
  decisão de infraestrutura pelo `sre` (réplica/HA do Redis), fora do escopo deste TRD.
- **Disponibilidade**: impacto **assimétrico** entre as duas metades da feature. Se `GET /:code`
  ficar indisponível, **todo link já compartilhado por qualquer pessoa, em qualquer lugar da
  internet, para de funcionar** — impacto crítico e externo (o PRD já nota isso na "Ordem de
  valor", F-3: "fecha o ciclo de utilidade"). Se `POST /api/links` ficar indisponível, o impacto é
  "não consigo criar um link novo agora" — degradação visível, mas sem quebrar links já
  distribuídos. Este TRD não propõe estratégia de resiliência diferenciada entre os dois endpoints
  (ambos dependem do mesmo Redis), mas sinaliza a assimetria para o `sre` priorizar monitoramento
  de `GET /:code` como o indicador de disponibilidade mais crítico da feature.
- **Observabilidade**: log estruturado nas duas fronteiras de adapter de entrada — em
  `CreateShortLinkHttpController`: código gerado, número de tentativas até sucesso (detecta
  degradação da taxa de colisão, `docs/adr/0004-...md`), resultado (sucesso/erro e qual código de
  erro); em `RedirectHttpController`: código requisitado, encontrado ou não, latência da consulta.
  Métricas mínimas: contagem de criações bem-sucedidas, contagem de colisões por tentativa,
  contagem de `RATE_LIMITED`, contagem de redirecionamentos bem-sucedidos vs. "não encontrado",
  latência (p50/p95) de `GET /:code` — estas métricas alimentam diretamente as "Métricas de
  sucesso" que o PRD já define na seção 8 (taxas de sucesso de encurtamento e de redirecionamento).
  Se o `sre`, ao revisar esta spec, instruir confirmar algum desses eventos em log de produção:
  registre aqui que, se não houver ocorrência recente dentro da janela de retenção de log
  disponível desta stack, isso vira pendência "VALIDAR DEPOIS" para a próxima ocorrência real, em
  vez de bloquear a aprovação daquela revisão.
- **Manutenibilidade**: nenhum desvio de SOLID/Clean Code.
- **Impacto em infraestrutura para o `sre` revisar**: precisa de uma instância Redis dedicada
  (armazenamento primário, não cache descartável — dimensionamento de memória/disco e estratégia
  de backup/HA relevantes, diferente de um cache que pode ser esvaziado sem perda permanente);
  precisa de um domínio público real para `SHORT_URL_BASE_DOMAIN` (o wireframe usa `short.ly`
  como placeholder — o domínio real de produção é uma decisão de produto/infra, não resolvida
  neste TRD); precisa da tarefa de CI descrita na seção 13 (primeira suíte de testes do projeto).
- **Indicadores técnicos herdados do PRD** (seção 9 do `prd.md`):
  - **Volumetria**: decisão tomada — Redis como armazenamento primário O(1) + geração de código
    aleatória com retry limitado em colisão (`docs/adr/0003-...md`, `docs/adr/0004-...md`).
  - **Segurança**: decisão tomada para v1, com mitigação deliberadamente parcial, não abrangente
    — validação de esquema da URL de destino (só `http(s)`) e rate limiting básico por IP
    (`docs/adr/0005-...md`), ambos **confirmados pelo usuário** (TRD-2 e TRD-3, seção 15).
    Moderação de reputação de destino fica fora de v1 deliberadamente, repassada ao
    `security-engineer` para aprofundar na revisão de segurança desta spec.
  - **Legal/compliance**: nenhuma decisão necessária — confirmado que não há dado pessoal
    coletado (uso anônimo, sem conta).

## 11. Plano de testes (alto nível)

- **Unitário**: os três value objects/entidade de domínio (`OriginalUrl`, `ShortCode`,
  `ShortLink`) têm teste próprio cobrindo toda entrada que deveria ser aceita e toda que deveria
  ser rejeitada pela invariante correspondente. Os dois casos de uso (`CreateShortLink`,
  `RedirectToOriginalUrl`) são testados com dublês de cada port que consomem (via
  `pytest-asyncio`, já que os ports e casos de uso são `async`) — nunca contra Redis real nesta
  camada. O comportamento de retry em colisão de `CreateShortLink` é verificado
  trocando a ordem/sequência de respostas do dublê de `ShortLinkWriterPort` (nunca por contagem
  fixa de tentativas no texto do teste — o número de tentativas é um detalhe de implementação, o
  que importa é que uma colisão reportada pelo port nunca derruba a criação enquanto houver
  tentativas restantes, e que o limite eventualmente desiste em vez de repetir para sempre).
- **Integração**: `RedisShortLinkRepository` e `RedisFixedWindowRateLimiter` contra um Redis real
  (container efêmero via `testcontainers-python`, `docs/STACK.md`) — cobrindo especificamente o
  comportamento atômico de `SET NX` sob escrita concorrente de dois códigos iguais (o cenário que
  a unidade, com dublê, não consegue provar) e a expiração real da janela de rate limit.
- **E2E**: esta é a primeira spec do projeto, e `docs/STACK.md` ainda não documenta nenhum harness
  de navegador real — o "e2e do fluxo humano" de cada fatia (seção 13) é verificado por integração
  HTTP encadeada entre os endpoints reais do backend, sem dublê (requisição real a
  `POST /api/links` seguida de requisição real a `GET /:code` com o código retornado, por
  exemplo), não por automação de navegador. Isso é uma decisão explícita desta spec, a reavaliar
  se um projeto futuro introduzir esse harness.
- **Frontend**: o componente de tela única (`CardEncurtador`) é testado contra um dublê de
  `LinkService` que respeita o contrato da seção 8 — toda transição de estado que o PRD descreve
  (ocioso, carregando, resultado exibido, erro de campo vazio, erro de URL inválida, confirmação de
  cópia) tem a asserção correspondente de o que fica visível/invisível na tela para aquele estado,
  nunca uma lista fechada de "quais textos aparecem" que precisaria ser reescrita a cada ajuste de
  copy. `LinkService` em si é testado contra um stub HTTP que simula as respostas reais do
  contrato (sucesso e cada código de erro).
- **Meta de cobertura**: 80% por pacote (`src/` e `frontend/`, cada um com seu próprio gate).

## 12. Riscos e trade-offs

| Risco | Impacto | Decisão / mitigação | Evidência |
|--------|-----------|-------------------------|--------------|
| Colisão de dois códigos aleatórios gerados simultaneamente para URLs diferentes | Sem tratamento atômico, a segunda escrita sobrescreveria a primeira (URL original trocada silenciosamente para quem já recebeu o link antigo) | `SET shortlink:<code> <url> NX` — a escrita só é efetivada se a chave ainda não existir; se já existir, `CreateShortLink` gera outro código e tenta de novo (até 5 tentativas, `docs/adr/0004-...md`) | `[não medido]` — atomicidade de `SET ... NX` é garantida pela documentação do Redis (execução de comando único serializada), a confirmar por teste de integração real contra a versão exata do Redis usada (escrita concorrente de dois clientes para o mesmo código), não só por leitura de documentação |
| Abuso de criação em massa por um único cliente anônimo | Esgotamento de espaço de código, custo de armazenamento, uso do serviço como gerador de links de phishing em escala | Rate limiting por IP, janela fixa de 60s (`docs/adr/0005-...md`) — explicitamente **não** mitiga um atacante distribuído em múltiplos IPs; aceito como trade-off de v1 | `[não medido]` — comportamento de `INCR`+`EXPIRE` como contador atômico é documentação do Redis, a confirmar com teste de integração que dispara requisições concorrentes do mesmo IP |
| Redirecionamento para destino malicioso (phishing/malware) | Dano de reputação ao domínio do encurtador, dano a terceiros que clicam no link | v1 só valida esquema (`http`/`https`); nenhuma verificação de reputação de conteúdo (`docs/adr/0005-...md`) — risco aceito conscientemente para v1, repassado ao `security-engineer` e ao usuário (pendência TRD-2, seção 15) | não aplicável (decisão de escopo, não afirmação sobre biblioteca/runtime) |
| Redis é ponto único de falha (sem fallback de leitura, sem réplica desenhada neste TRD) | Indisponibilidade total de criação **e** de redirecionamento se a instância cair | TRD não desenha HA (decisão de infraestrutura, não de arquitetura de aplicação) — sinalizado ao `sre` na seção 10 | não aplicável (decisão de escopo) |
| Comportamento do `redis.asyncio` (`redis-py`) ao perder conexão a meio de um comando `SET NX`/`GET` | Se a biblioteca reenviar automaticamente o comando após reconectar, um `try_save` poderia, em teoria, ser executado duas vezes (idempotente neste caso específico, porque `NX` torna a segunda tentativa um no-op seguro) — mas isso não foi verificado | Nenhuma mitigação adicional além de `NX` já ser idempotente por construção para o caso de escrita; para leitura (`GET`), reenvio automático não tem efeito colateral (é uma consulta) | `[não medido]` — comportamento de reconexão/retry do `redis-py` (versão a fixar pelo `backend-developer` em `pyproject.toml`) não foi testado nesta rodada; a verificar por quem implementar, com teste de integração que force desconexão a meio de uma operação |

## 13. Decomposição de tarefas e dependências (fatias verticais de entrega)

Nenhuma tarefa desta spec depende da forma concreta de um contrato externo não observado (passo
1c do processo do `architect`) — todo o sistema (backend, frontend, Redis) é construído do zero
nesta mesma spec, sem integração com DOM/API/arquivo de terceiro. Nenhuma tarefa de investigação é
necessária por esse motivo.

Remote GitHub configurado (`https://github.com/asengardeon/encurtador-url`, privado) — as 13
issues abaixo foram criadas, todas com milestone `0001-encurtar-url` e label de trilha
(`backend`/`frontend`/`ambos`) espelhando a coluna "Trilha", além do label `enhancement`.

| ID   | Tarefa | Trilha | Fatia (PRD) | Depende de | Validação na fatia | Status | Issue GitHub |
|------|----------|----------|---------------|---------------|------------------------|-----------|------------------|
| T-1  | Criar `.github/workflows/ci.yml` conforme a forma mínima de `docs/CI-WORKFLOW.md`, usando os comandos de lint/teste/cobertura de `docs/STACK.md` (jobs para `src/` e `frontend/`) | ambos | F-1 | nenhuma | sre (obrigatório — tarefa toca `ci.yml`) | pendente | #1 |
| T-2  | Domínio backend: `OriginalUrl`, `ShortCode`, `ShortLink` com testes unitários cobrindo toda entrada válida/invalida de cada invariante (seção 4) | backend | F-1 | nenhuma | — | pendente | #2 |
| T-3  | Ports de escrita (`ShortLinkWriterPort`, `ShortCodeGeneratorPort`, `RateLimiterPort`) e caso de uso `CreateShortLink` (conforme US-1, seção 6), testados com dublês dos três ports | backend | F-1 | T-2 | segurança — antecipada por julgamento do `architect`: esta tarefa implementa as duas mitigações de v1 que o PRD sinaliza como indicador de segurança (validação de esquema da URL e rate limiting, `docs/adr/0005-...md`), relevante verificar antes do fim da spec | pendente | #3 |
| T-4  | Adapters de saída: `RedisShortLinkRepository.try_save`, `Base62RandomShortCodeGenerator`, `RedisFixedWindowRateLimiter` — testados em integração contra Redis real (`testcontainers-python`, `docs/STACK.md`) | backend | F-1 | T-3 | — | pendente | #4 |
| T-5  | Adapter de entrada `CreateShortLinkHttpController` (`POST /api/links`, conforme contrato da seção 8) + bootstrap do app FastAPI/Uvicorn + variáveis de ambiente novas (`REDIS_URL`, `SHORT_URL_BASE_DOMAIN`, `RATE_LIMIT_CREATE_MAX_PER_MINUTE`) adicionadas a `.env.example` e a um validador de ambiente do backend (`pydantic-settings`) — critério de aceite explícito desta tarefa | backend | F-1 | T-4 | — | pendente | #5 |
| T-6  | Frontend: componente `CardEncurtador` (wireframe Opção A, `docs/DESIGN-SYSTEM.md`) cobrindo os estados ocioso/carregando/resultado/erro-campo-vazio/erro-URL-inválida de US-1, consumindo `LinkService` (dublê do contrato da seção 8 até T-5 existir) | frontend | F-1 | nenhuma | — | pendente | #6 |
| T-7  | E2E da fatia F-1: integração HTTP encadeada contra o backend real (conforme seção 11) cobrindo os cinco cenários de US-1, incluindo "encurtar a mesma URL duas vezes gera código novo" | backend | F-1 | T-5, T-6 | — | pendente | #7 |
| T-8  | Frontend: botão "Copiar" (US-2) com Clipboard API e confirmação visual, sobre o resultado já exibido por T-6 | frontend | F-2 | T-6 | — | pendente | #8 |
| T-9  | Teste frontend do fluxo "encurtar depois copiar", com dublê da Clipboard API (sem harness de navegador real — mesma decisão da seção 11) | frontend | F-2 | T-6, T-8 | — | pendente | #9 |
| T-10 | Backend: `ShortLinkReaderPort` + caso de uso `RedirectToOriginalUrl` (US-3, seção 6), testado com dublê do port | backend | F-3 | T-2 | — | pendente | #10 |
| T-11 | Backend: `RedisShortLinkRepository.find_by_code` (completa o adapter de T-4), testado em integração contra Redis real | backend | F-3 | T-4, T-10 | — | pendente | #11 |
| T-12 | Backend: `RedirectHttpController` (`GET /:code`, contrato da seção 8), incluindo a página HTML estática de "não encontrado" | backend | F-3 | T-11 | segurança (obrigatório — ponto de entrada que aceita um identificador externo controlado pelo requisitante, código curto, e o usa para decidir o destino de um redirecionamento; é exatamente o risco de redirecionamento aberto que o PRD sinaliza na seção 9) | pendente | #12 |
| T-13 | E2E da fatia F-3: integração HTTP encadeada — cria um link via o endpoint de T-5 (já mergeado em F-1) e acessa o código retornado via T-12, confirmando `302`/`Location`; acessa um código inexistente e confirma `404` | backend | F-3 | T-12 | — | pendente | #13 |

Checagem do roteiro de cada e2e contra a fatia que o declara (passo 8b do processo do
`architect`): T-7 (F-1) não tem pré-condição externa — é o primeiro ponto de entrada da feature,
uso anônimo sem login. T-13 (F-3) pressupõe "um link encurtado já existe"; o produtor dessa
pré-condição é o endpoint de T-5, publicado na fatia F-1, já mergeada antes de F-3 começar (mesma
ordem de dependência que o PRD já declara na seção 10, "F-3 depende de F-1") — produtor alcançável,
nenhuma lacuna.

### Janelas de quebra de contrato entre fatias

Não aplicável. As três fatias desta spec são estritamente aditivas: F-1 introduz
`POST /api/links` (endpoint novo, nada pré-existente para quebrar), F-2 é só frontend sobre um
resultado já exibido, F-3 introduz `GET /:code` (endpoint novo). Nenhuma fatia torna um campo
obrigatório numa rota já consumida por um cliente implantado, porque não existe cliente implantado
antes desta spec — é a primeira feature do projeto.

## 14. Controle de versão (GitHub Flow, por fatia)

| Fatia | Branch | PR | Status |
|-------|----------|------|----------|
| F-1   | `feature/0001-encurtar-url/f-1` | (criado quando `/btt-sdd:implement` iniciar a fatia) | não iniciado |
| F-2   | `feature/0001-encurtar-url/f-2` | (criado quando `/btt-sdd:implement` iniciar a fatia) | não iniciado |
| F-3   | `feature/0001-encurtar-url/f-3` | (criado quando `/btt-sdd:implement` iniciar a fatia) | não iniciado |

Cada fatia é full-stack (F-1 e F-3 têm tarefas de backend e frontend na mesma branch, árvores
`src/`+`tests/` vs. `frontend/` evitando conflito de merge; F-2 é só frontend). Nenhuma fatia
carrega `!` de breaking change no título do PR — todas são aditivas (seção 13).

## 15. Pendências de validação (VALIDAR DEPOIS)

| ID | Pergunta | Contexto | Status |
|------|------------|------------|----------|
| TRD-1 | A stack proposta estava confirmada, ou o usuário quer ajustar algum componente? | Alimentava a seção 2 deste TRD e `docs/STACK.md` por completo. **Resolvida em 2026-10-31**: backend corrigido de Node.js/Fastify para Python/FastAPI assíncrono (motivo do usuário: encaixe com alto tráfego + Redis assíncrono); frontend e Redis confirmados sem alteração. `docs/adr/0002-stack-tecnologica-encurtador-url.md`. | validado |
| TRD-2 | A mitigação v1 de redirecionamento aberto (só validar esquema `http`/`https`, sem verificação de reputação de destino) é aceitável para o lançamento inicial? | Decisão de produto/segurança — alimenta a revisão de segurança antecipada nas tarefas T-3/T-12. **Resolvida em 2026-10-31: confirmada sem alteração.** `docs/adr/0005-mitigacao-v1-abuso-e-redirecionamento-aberto.md`. | validado |
| TRD-3 | O valor padrão de `RATE_LIMIT_CREATE_MAX_PER_MINUTE` (proposto: 20) é adequado? | Número de produto — afeta a tarefa T-5. **Resolvida em 2026-10-31: confirmado 20 requisições/minuto por IP, sem alteração.** | validado |
| TRD-4 | O usuário vai criar um repositório novo no GitHub, ou já existe um para apontar como remote? | Bloqueava toda a seção 13. **Resolvida em 2026-10-31**: repositório novo criado e configurado como remote `origin` — `https://github.com/asengardeon/encurtador-url` (privado). As 13 issues foram criadas (milestone `0001-encurtar-url`). | validado |
| TRD-5 | Qual é o domínio real de produção para `SHORT_URL_BASE_DOMAIN` (o wireframe usa `short.ly` como placeholder de design, não uma decisão de infraestrutura)? | Não bloqueia a implementação (o valor de desenvolvimento/teste pode ser qualquer placeholder, ex. `localhost:8000`), mas precisa existir antes do primeiro deploy real — mais relevante para o `sre` que para este TRD. Ainda sem resposta do usuário. | pendente |

## 16. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|
| 2026-10-30 | agente `architect` | Criação deste TRD a partir do PRD aprovado de `0001-encurtar-url` | Primeiro desenho técnico da feature | nenhuma (primeira versão) |
| 2026-10-31 | agente `architect` | Backend trocado de Node.js/Fastify para Python/FastAPI assíncrono (seções 1, 2, 4, 5, 6, 7, 9, 10, 11, 12); TRD-1/TRD-2/TRD-3/TRD-4 resolvidas (seção 15); remote GitHub configurado e as 13 issues de tarefa criadas, coluna "Issue GitHub" da seção 13 preenchida (`#1`–`#13`); `docs/STACK.md` e os quatro ADRs atualizados para refletir as respostas do usuário | Respostas do usuário às pendências levantadas na primeira versão deste TRD | Seções 1, 2, 4, 5, 6, 7, 9, 10, 11, 12, 13, 15 revalidadas contra as respostas; demais seções inalteradas |

## 17. Aprovação

- [x] Aprovado por: usuário em 2026-10-31. TRD-5 (domínio de produção) permanece como pendência
  não bloqueante — a resolver antes do primeiro deploy, não antes da implementação.
