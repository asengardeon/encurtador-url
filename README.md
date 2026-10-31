# encurtador-url

Projeto criado com um template de desenvolvimento orientado a especificação (**SDD —
Spec-Driven Development**) para o Claude Code: cada feature nasce de um PRD, passa por um TRD,
é implementada com TDD em ports & adapters (backend e frontend em paralelo quando aplicável), é
validada por QA e segurança, e liberada por SRE — com um agente dedicado a cada etapa.

Comece por `CLAUDE.md` — é o arquivo que o Claude Code lê automaticamente e que explica todo o
pipeline. Para a explicação de cada arquivo/pasta, veja `FILE-GUIDE.md` do pipeline (bundled com
os agentes/skills, não copiado para dentro deste projeto — ver `CLAUDE.md` para onde encontrá-lo).

## Pipeline

```
/sdd-prd  →  /sdd-trd  →  /sdd-implement  →  /sdd-qa  →  /sdd-security  →  /sdd-sre
```

Use `/sdd-status` a qualquer momento para ver em que etapa cada feature está, e `/sdd-pending`
para ver o que ainda precisa de validação.

## Ferramentas recomendadas

Instale [rtk](https://github.com/rtk-ai/rtk) (uma vez por máquina) para reduzir o consumo de
tokens dos comandos de shell que este pipeline roda com frequência — ver `TOOLING.md` do pipeline.

## Estado atual

Este projeto ainda não tem stack, código, nem infraestrutura definidos — foi criado agora por
`/create-project`. O primeiro PRD está em `specs/0001-<slug>/prd.md`. Próximo passo: aprove-o e
rode `/sdd-trd`.
