# ADR 0002 — Adotar Python/FastAPI (assíncrono) + Redis + Vite para o encurtador de URL

> Status: aceito (confirmado pelo usuário em 2026-10-31, com correção em relação à proposta
> original do `architect` — ver "Decisão" e "Log")

## Contexto

Este é o primeiro TRD do projeto `encurtador-url`. Não existia `docs/STACK.md` nem
`~/.claude/stack-defaults.md` prévios, então a stack precisou ser decidida do zero
(`.claude/agents/architect.md`, passo 2). O PRD (`specs/0001-encurtar-url/prd.md`, seção 9) exige
considerar uso público de alto tráfego, e a feature é full-stack (uma tela única colando URL +
backend de geração/persistência/redirecionamento).

O `architect` propôs inicialmente TypeScript/Node.js + Fastify para o backend (ver "Log" abaixo).
O usuário, ao revisar a pendência TRD-1, não confirmou essa parte da proposta e pediu Python +
FastAPI assíncrono em vez disso.

## Decisão

- **Backend**: Python 3.12+, framework FastAPI, servido por Uvicorn (ASGI) — **assíncrono de
  ponta a ponta**: rotas `async def`, cliente Redis assíncrono (`redis.asyncio`, pacote
  `redis-py`). Motivo dado pelo usuário: FastAPI assíncrono encaixa melhor com o indicador de alto
  tráfego do PRD — o caminho de redirecionamento (`GET /:code`) não bloqueia o event loop
  esperando a resposta do Redis, permitindo que uma única instância do processo atenda muitas
  requisições concorrentes de I/O sem precisar de um pool de threads.
- **Frontend**: TypeScript vanilla com Vite, sem framework de componentização — **sem alteração**
  em relação à proposta original.
- **Persistência**: Redis — **sem alteração** em relação à proposta original (decisão detalhada
  em ADR 0003).
- **Testes**: `pytest`/`pytest-asyncio`/`pytest-cov` (backend); Vitest + cobertura `v8`
  (frontend).

## Alternativas consideradas

- **Node.js + Fastify para o backend (proposta original do `architect`)**: também é assíncrono
  por natureza (event loop único do Node) e teria entregado a mesma vantagem de não bloquear em
  I/O de Redis, compartilhando linguagem com o frontend (TypeScript nos dois lados). Rejeitada
  pelo usuário em favor de Python/FastAPI — a motivação registrada foi a combinação específica
  FastAPI assíncrono + Redis assíncrono, não uma limitação técnica do Node/Fastify.
- **Go** para o backend: desempenho de redirecionamento provavelmente superior ao de
  Python/FastAPI para o caminho mais quente da feature, mas introduziria uma segunda linguagem no
  projeto (frontend continua em TypeScript/JavaScript por rodar no navegador). Não escolhida.
- **FastAPI síncrono** (rotas `def`, cliente Redis síncrono): mais simples de escrever, mas exige
  um pool de threads (via `run_in_threadpool`) para não bloquear o event loop do Uvicorn sob
  I/O concorrente — perde exatamente a vantagem que motivou a escolha de FastAPI. Rejeitada.
- **React/Vue no frontend**: avaliado e rejeitado para esta v1 — uma única tela com poucos estados
  dinâmicos (idle, carregando, resultado, erro) não justifica o custo de um framework de
  componentização; TypeScript vanilla + Vite entrega o mesmo resultado com menos superfície.
  Reavaliar se uma spec futura expandir a UI. Não afetado pela troca de backend.
- **Banco relacional (Postgres) como armazenamento principal**: avaliado e rejeitado — ver ADR
  0003.

## Consequências

- Backend (Python) e frontend (TypeScript) **não compartilham linguagem** — diferente da proposta
  original, que mantinha os dois em TypeScript. Nenhum tipo é compartilhado automaticamente entre
  os dois lados; cada um declara seus próprios tipos contra o contrato da seção 8 do TRD
  (Pydantic no backend, interfaces TypeScript no frontend). Isso já era o plano mesmo na proposta
  original (sem pacote `shared/`), então o custo adicional de duas linguagens é só o de alternar
  contexto entre elas, não o de sincronizar tipos manualmente de um jeito novo.
- FastAPI/Uvicorn e Redis assíncrono (`redis.asyncio`) são ambos otimizados para o caminho de
  leitura/escrita de alto volume que o PRD sinaliza, de ponta a ponta assíncrono — a escolha
  final prioriza esse indicador técnico com a stack que o próprio usuário considerou mais
  adequada.
- `mypy` passa a fazer parte do gate de lint do backend (ausente numa stack puramente TypeScript,
  onde o compilador já checa tipos) — decisão de manutenibilidade registrada em `docs/STACK.md`.
- Esta decisão está **confirmada** — não há mais pendência de stack em aberto (TRD-1 resolvida).

## Log

- 2026-10-30: proposto pelo `architect` (TypeScript/Node.js/Fastify), pendente de confirmação do
  usuário (invocação sem acesso a `AskUserQuestion`).
- 2026-10-31: usuário respondeu TRD-1 — backend corrigido para Python/FastAPI assíncrono; demais
  componentes (frontend, Redis) confirmados sem alteração. Status passa a "aceito".
