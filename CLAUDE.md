# Guia para o Claude Code

Este projeto usa **SDD — Spec-Driven Development**, o pipeline do agente/skill `btt-sdd`: toda
feature nasce de uma especificação de produto, passa por um desenho técnico revisável, é
implementada com TDD e só chega a produção depois de revisão de código e QA — com revisão de UX,
revisão de segurança e validação de SRE ao final de cada spec, sobre o diff acumulado (ou
antecipadas para uma fatia específica quando a decomposição de tarefas do TRD declara isso).

**Os agentes, skills e docs de governança genéricos deste pipeline (`GIT-WORKFLOW.md`,
`QUALITY-GATES.md`, `TESTING.md`, `CI-WORKFLOW.md`, `ENGINEERING-PILLARS.md`, `ARCHITECTURE.md`,
`SDD-WORKFLOW.md`, `FILE-GUIDE.md`, `POST-MERGE-VALIDATION.md`, `TOOLING.md`) não vivem neste
projeto** — vêm do plugin `btt-sdd` instalado (`claude plugin install`) ou da distribuição global
via junction, e se mantêm sempre atualizados sozinhos (`claude plugin update`, ou live via
junction). Não há cópia local desses docs para sincronizar. O `docs/` deste projeto só tem
conteúdo que é genuinamente dele: `STACK.md` (decisão de stack, criada pelo `architect` no primeiro
TRD), `BASELINE.md` (condicional), `PROJECT-CONVENTIONS.md` (condicional — particularidades deste
projeto em relação ao padrão genérico do pipeline: git workflow, estrutura de pastas etc.),
`LESSONS-LEARNED.md` (condicional) e `adr/` (decisões de arquitetura específicas deste projeto).

**Recomendado**: instale [rtk](https://github.com/rtk-ai/rtk) (uma vez por máquina, não por
projeto) para reduzir o consumo de tokens dos comandos de shell que este pipeline roda com
frequência (testes, lint, Git) — ver `TOOLING.md` do pipeline para o porquê e o passo a passo.
Nenhuma etapa deste pipeline bloqueia por rtk ausente; é só uma economia de custo.

Comandos disponíveis: `/sdd-prd`, `/sdd-trd`, `/sdd-implement`, `/sdd-code-review`,
`/sdd-ux-review` (condicional — ao final da spec, se ela tem superfície de UI perceptível),
`/sdd-qa`,
`/sdd-security`, `/sdd-sre`, `/sdd-baseline`, `/sdd-project-conventions`, `/sdd-hotfix`,
`/sdd-status`, `/sdd-amend`, `/sdd-pending`, `/sdd-gap-report` (ou os equivalentes `/btt-sdd:*` se
instalado via plugin). Cada um explica seu próprio papel quando invocado — não precisa memorizar a
lista aqui.

**As etapas 4-7 e 4b (revisão) sempre passam pela skill própria — nunca invoque `code-reviewer`,
`ux-designer`, `qa-engineer`, `security-engineer` ou `sre` diretamente via Agent tool fora de
`/sdd-code-review`/`/sdd-ux-review`/`/sdd-qa`/`/sdd-security`/`/sdd-sre` (ou os equivalentes
`/btt-sdd:*`) — mesmo numa sessão que já rodou esse pipeline inteiro para uma fatia anterior.** A
tentação de pular direto para o agente cresce com a familiaridade ("já sei os passos"), mas é a
skill que carrega o gate de pré-condição de cada etapa (ex.: QA aprovado antes de segurança) —
pular a skill não pula o gate em si (o agente de revisão recusa corretamente quando a
pré-condição falha), mas desperdiça uma invocação inteira só para descobrir isso. Isso é
diferente da etapa 3 (`backend-developer`/`frontend-developer`), onde a invocação direta via
Agent tool é o padrão correto — `/sdd-implement` não tem um agente intermediário.

## Próximo passo

Este projeto acabou de ser criado por `/create-project` — ainda não tem stack, código, nem
infraestrutura definidos. O primeiro PRD deve estar em `specs/0001-<slug>/prd.md`; aprove-o e
rode `/sdd-trd` para o `architect` desenhar a solução técnica (é aí que a stack é decidida e
`docs/STACK.md` nasce).
