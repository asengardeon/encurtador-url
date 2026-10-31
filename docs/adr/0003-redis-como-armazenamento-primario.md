# ADR 0003 — Redis como armazenamento primário do mapeamento código→URL

> Status: aceito (confirmado pelo usuário em 2026-10-31, sem alteração em relação à proposta
> original — ver `docs/STACK.md`)

## Contexto

O PRD (`specs/0001-encurtar-url/prd.md`, seção 9, indicador "Volumetria") exige considerar a
escala de leitura/escrita do mapeamento código→URL sob uso público de alto tráfego. A mesma spec
exclui explicitamente contas, histórico, deduplicação e expiração (seção 5, "Fora de escopo") —
não há necessidade de consulta relacional (JOIN, filtro por usuário, filtro por data) nesta v1. O
domínio de dados é, literalmente, um par `código → URL original`, sempre permanente enquanto esta
spec estiver ativa.

## Decisão

Redis é o armazenamento **primário** (não um cache em frente a outro banco) do mapeamento
código→URL. Leitura (`GET`) no caminho de redirecionamento e escrita atômica (`SET ... NX`) no
caminho de criação são ambas operações O(1) de baixa latência, adequadas ao indicador de
volumetria do PRD. Persistência habilitada via AOF (`appendonly yes`) para que o mapeamento
sobreviva a um reinício do processo Redis — sem isso, todo link gerado se perderia a cada restart,
o que violaria o critério de aceite de US-3 ("link encurtado válido foi gerado anteriormente").

## Alternativas consideradas

- **PostgreSQL (ou outro banco relacional) como armazenamento principal, com Redis como cache em
  frente**: arquitetura mais robusta para um domínio que crescesse (histórico, deduplicação,
  métricas de clique visíveis), mas nenhum desses requisitos existe nesta spec — introduzir dois
  armazenamentos (e a complexidade operacional de mantê-los consistentes) seria over-engineering
  para o escopo atual. Rejeitada por desproporção entre complexidade e requisito real.
- **Redis só como cache, com Postgres como fonte de verdade**: mesma rejeição acima — sem
  necessidade de consulta relacional, não há "fonte de verdade" adicional a justificar.
- **Arquivo/SQLite local**: descartado por não escalar horizontalmente (múltiplas instâncias do
  backend não podem compartilhar um arquivo local sem um serviço de coordenação por cima, o que
  anularia a vantagem de simplicidade).

## Consequências

- Simplicidade operacional: um único serviço com estado a gerenciar, não dois.
- Redis passa a ser uma dependência crítica de disponibilidade (sem Redis, nem criação nem
  redirecionamento funcionam) — não há fallback de leitura possível, porque não existe segunda
  fonte. Risco registrado no TRD (`specs/0001-encurtar-url/trd.md`, seção 12) e sinalizado para o
  `sre` avaliar réplica/alta disponibilidade em produção, fora do escopo deste TRD.
- Se uma spec futura (ex.: `0002-expirar-link`, ou uma eventual necessidade de histórico/métricas
  visíveis) precisar de consulta relacional, esta decisão é revisitada com um novo ADR — não
  editada aqui.
- Esta decisão está **confirmada pelo usuário** (`docs/STACK.md`, nota de status) — a troca de
  backend para Python/FastAPI (ADR 0002) não altera nada aqui: o cliente Redis usado é
  `redis.asyncio` (pacote `redis-py`) em vez de `ioredis`, mas a decisão de Redis como
  armazenamento primário é independente da linguagem do backend.
