# Stack Tecnológica

Gerado/mantido pelo agente `architect` (processo em `.claude/agents/architect.md`). É a fonte que
qualquer TRD novo neste projeto reaproveita para não perguntar a stack de novo.

> **Status desta decisão: confirmada pelo usuário em 2026-10-31**, com uma correção em relação à
> proposta inicial do `architect` (TRD-1, `specs/0001-encurtar-url/trd.md`, seção 15): o backend
> passa a ser **Python + FastAPI**, não Node.js/Fastify. Frontend (TypeScript vanilla + Vite) e
> Redis 7.x como armazenamento primário permanecem exatamente como propostos. Motivo da troca,
> dado pelo próprio usuário: FastAPI assíncrono encaixa melhor com o indicador de alto tráfego do
> PRD e com um cliente Redis assíncrono, sem bloquear o event loop no caminho de
> leitura/escrita mais exercitado da feature (redirecionamento).

- **Linguagem/runtime**: Python 3.12+ no backend; TypeScript sobre Node.js 22 LTS só como
  toolchain de build do frontend (o runtime de produção do frontend é o navegador, não Node —
  Node entra apenas para rodar Vite/TypeScript em tempo de desenvolvimento/build).
- **Framework principal (backend)**: FastAPI, servido por Uvicorn (ASGI), **assíncrono de ponta a
  ponta** — rotas `async def`, cliente Redis assíncrono (ver abaixo). Essa escolha é a correção
  de TRD-1: evita bloquear o event loop por I/O de rede (Redis) sob o alto tráfego que o PRD
  sinaliza (seção 9), sem precisar de um pool de threads para paralelizar I/O como a alternativa
  síncrona exigiria.
- **Framework principal (frontend)**: nenhum framework de componentização — Vite + TypeScript
  vanilla. A feature é uma única tela com poucos estados (`docs/DESIGN-SYSTEM.md`,
  `specs/0001-encurtar-url/prd.md`, seção 7); um framework de UI completo (React/Vue) seria
  complexidade sem retorno para este escopo. Reavaliar nesta decisão se uma spec futura expandir a
  superfície de UI além de uma tela única. Não afetado pela troca de backend.
- **Persistência**: Redis 7.x como armazenamento primário (não só cache) do mapeamento
  código→URL — decisão e trade-off detalhados em
  `docs/adr/0003-redis-como-armazenamento-primario.md`. Cliente Python: `redis-py` usando sua
  interface assíncrona (`redis.asyncio`), coerente com o FastAPI assíncrono acima. Persistência
  habilitada via AOF (`appendonly yes`) para sobreviver a reinícios, já que não há banco
  relacional nesta v1.
- **Gerenciador de pacotes**:
  - Backend: `pip` + `pyproject.toml` (build backend `setuptools`).
  - Frontend: `npm`, projeto independente em `frontend/` com seu próprio `package.json`/lockfile.
  - Sem workspace/pacote compartilhado entre os dois: o contrato de API (TRD, seção 8) é pequeno o
    suficiente para cada lado declarar seus próprios tipos (Pydantic no backend, interfaces
    TypeScript no frontend) sem um pacote `shared/`. Reavaliar como ADR se a duplicação crescer em
    specs futuras.
- **Testes/cobertura (backend)**: `pytest` + `pytest-asyncio` (rotas/casos de uso assíncronos) +
  `pytest-cov`, gate de 80%.
- **Testes/cobertura (frontend)**: [Vitest](https://vitest.dev/) + provider de cobertura `v8`,
  gate de 80%.
- **Lint/tipos (backend)**: `ruff` (lint + formatação) + `mypy` (checagem de tipo estática —
  relevante para a camada de ports/domínio, onde o contrato de tipos entre caso de uso e adapter é
  parte do design, não só estilo).
- **Lint (frontend)**: ESLint + `typescript-eslint`.
- **Validação de variáveis de ambiente (backend)**: `pydantic-settings` — schema explícito que
  falha rápido (na inicialização do processo) se uma variável obrigatória estiver ausente, em vez
  de falhar só na primeira requisição que precisar dela.
- **Proveniência desta decisão**: stack original proposta pelo `architect` sem acesso a
  `AskUserQuestion` nesta sessão; o backend foi corrigido pelo usuário (TRD-1) antes da aprovação
  do TRD — ver `docs/adr/0002-stack-tecnologica-encurtador-url.md` para a decisão final e as
  alternativas consideradas (incluindo a proposta original rejeitada, Node.js/Fastify).

## Execução de testes (comandos e isolamento)

- **Unitário**:
  - Backend: `pytest -m unit` (ou diretório `tests/unit/` dedicado) — usa dublês dos ports via
    `unittest.mock`/fakes escritos à mão, nunca Redis real. Seguro para paralelizar
    (`pytest-xdist`, `-n auto`).
  - Frontend: `npm run test:unit` → `vitest run --project unit`, mesmo critério de dublê do
    `LinkService`. Seguro para paralelizar (`--pool=threads`, padrão do Vitest).
- **Integração** (`pytest -m integration`, backend): exercita os adapters reais
  (`RedisShortLinkRepository`, `RedisFixedWindowRateLimiter`) contra um Redis real.
  - **Isolamento entre execuções concorrentes** (worktrees/agentes paralelos,
    `docs/GIT-WORKFLOW.md`, "Isolamento de arquivos não é isolamento de serviços com estado"):
    cada execução da suíte de integração sobe seu **próprio container Redis efêmero** via
    [`testcontainers-python`](https://testcontainers-python.readthedocs.io/) (imagem
    `redis:7-alpine`, porta de host aleatória alocada pela biblioteca) em vez de apontar para uma
    instância Redis compartilhada — não há contenção entre worktrees porque não há serviço
    compartilhado nenhum a disputar.
  - O container sobe uma vez por execução (fixture `pytest` de escopo `session`) e é removido ao
    final da sessão de teste (nunca deixado no ar — `docs/TESTING.md`, "Ambiente Docker de teste é
    efêmero por execução"). Cada teste individual reseta o estado com `FLUSHDB` entre casos
    (fixture de escopo `function`), em vez de recriar o container.
  - Os testes de integração rodam **sequencialmente dentro da mesma execução** (sem
    `pytest-xdist` nesta camada), porque compartilham o mesmo container — paralelizar aqui
    reintroduziria a mesma classe de corrida que o isolamento entre execuções evita.
- **Investigação/correção** (nunca na rodada final de evidência): `pytest -x` (backend),
  `vitest run --bail=1` (frontend).
- **Repórter condensado**: `pytest -q` (backend) / `--reporter=dot` (frontend, Vitest) para
  leitura humana/agente. Saída legível por máquina: `pytest --cov-report=json` (backend,
  `coverage.json`) e `vitest run --reporter=json --outputFile=<path>` (frontend) — cobertura
  nunca extraída do relatório HTML.
- **Cache de dependências**: backend — cache de `pip` chaveado por `pyproject.toml`/lockfile no
  CI; frontend — `actions/setup-node` com `cache: npm`, `cache-dependency-path` apontando para
  `frontend/package-lock.json`.

## Simulação de nuvem local (dev/teste)

Não aplicável a esta feature. Nenhum adapter de `0001-encurtar-url` fala com um serviço gerenciado
de nuvem específico (AWS S3/DynamoDB/SQS/Lambda, Azure Blob Storage, GCP Cloud Storage, OCI Object
Storage) — o Redis usado em desenvolvimento/teste é o motor real via container local
(`testcontainers-python`, ver acima), não uma simulação de API de nuvem. Esta seção permanece como
referência para quando uma feature futura introduzir esse tipo de dependência.

| Serviço de nuvem usado | Provedor emulado por floci | Registrado em |
|--------------------------|-------------------------------|------------------|
| _(nenhum ainda)_         | —                              | —                 |

## Proveniência

Decidida durante o TRD da feature `0001-encurtar-url`
(`specs/0001-encurtar-url/trd.md`), primeira feature deste projeto. Backend corrigido de
Node.js/Fastify para Python/FastAPI por decisão do usuário (TRD-1) antes da aprovação final do
TRD.

## Log de revisões

| Data       | Autor               | O que mudou           | Motivo                                                    |
|------------|------------------------|--------------------------|----------------------------------------------------------------|
| 2026-10-31 | agente `architect`    | Backend trocado de Node.js/Fastify para Python/FastAPI assíncrono; comandos de teste/lint atualizados para o ecossistema Python (`pytest`, `ruff`, `mypy`, `pydantic-settings`, `testcontainers-python`) | Resposta do usuário a TRD-1: FastAPI assíncrono encaixa melhor com alto tráfego + Redis assíncrono |
| 2026-10-30 | agente `architect`    | Criação deste arquivo  | Primeiro TRD do projeto (`0001-encurtar-url`) decide a stack. |
