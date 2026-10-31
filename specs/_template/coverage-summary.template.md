# Resumo de cobertura — <slug> / <fatia> / <trilha>

> Gerado por: `<backend-developer|frontend-developer|qa-engineer>`
> Branch: `<nome-da-branch>`
> Commit (último que tocou código **desta trilha**) **desta rodada**: `<saída de git log -1 --format=%H -- <caminhos de produção da trilha>>`
> Gerado em: `<data/hora>`
> Comando executado: `<comando de lint + teste/cobertura da stack em uso>`

Este arquivo é a **evidência condensada** de uma execução real da suíte completa com cobertura — não
um relatório bruto (nunca cole aqui o HTML/XML/JSON original, nem um dump linha-a-linha de arquivo
100% coberto). Extraia só o que muda uma decisão de aprovação: números agregados e as lacunas abaixo
do gate. Qualquer etapa seguinte que precise desta evidência **lê este arquivo em vez de rodar a
suíte de novo**, contanto que o campo `Commit` acima seja igual à saída atual de `git log -1
--format=%H -- <caminhos de produção desta trilha>` na branch do PR (`src/` para backend,
`frontend/` para frontend, no layout padrão, mais os caminhos compartilhados que a trilha consome).
**Nunca o HEAD literal** (`git rev-parse HEAD`),
que avança a cada commit de documentação das etapas de revisão e faria o artefato parecer
desatualizado sem nada de código ter mudado — e **nunca os caminhos das duas trilhas juntos**
(`-- src/ frontend/`), que fazem uma correção só de backend invalidar a evidência de frontend, e
vice-versa, forçando a reexecução mais cara do pipeline sem que o código daquela trilha tenha
mudado (ver `TESTING.md` do pipeline, seção "Reaproveitamento
do artefato de cobertura entre etapas"). Se divergir, quem precisar da evidência roda a suíte **desta
trilha** e regrava este arquivo — nunca segue com um número desatualizado, e nunca reexecuta a
trilha cujo `Commit` continua batendo.

**O cabeçalho acima é da rodada vigente, nunca histórico.** Toda rodada que regrava este arquivo —
inclusive a 2ª, 3ª e 4ª rodadas de correção de um mesmo PR — **atualiza** `Branch`, `Commit` e `Gerado
em` com os valores da própria rodada. Herdar o `Commit` da primeira rodada é pior do que não ter o
campo: é **desse** campo que o protocolo de reaproveitamento de evidência parte, então um SHA velho faz
a etapa seguinte reaproveitar como fresca uma execução que não cobre o código atual. Já aconteceu de
verdade: depois de três rodadas de correção, o cabeçalho ainda apontava para o commit da rodada 1 — pego
por uma rodada de code review, não por quem escreveu o arquivo.

**A base de qualquer diff é calculada, nunca lida de tabela de artefato.** Use sempre `git merge-base
HEAD main` (ou a branch base do projeto). O SHA que a coluna `Commit` das tabelas "Histórico de
aprovações por fatia" registra é capturado **antes** do merge daquela fatia; com *squash merge*
(`docs/GIT-WORKFLOW.md`) ele deixa de ser ancestral de `main` e, usado como base, produz um diff
gigante e errado. Já aconteceu de verdade: um `security-engineer` só revisou o diff certo porque apurou
a base por conta própria, depois de `merge-base` refutar o commit que os cabeçalhos "vigente" dos
artefatos nomeavam — se tivesse confiado na tabela, teria revisado outra fatia.

## Resultado da suíte

| Métrica              | Resultado |
|-----------------------|-----------|
| Testes                | `<N passaram, M falharam, K pulados>` |
| Lint                  | `<sem erros / N erros>` |
| Build/empacotamento   | `<sucesso / falhou — comando: <comando real de build/produção da stack, docs/STACK.md> | "não aplicável — trilha sem artefato de build/empacotamento próprio">` |

## Cobertura agregada (por pacote)

| Pacote     | Linhas | Branches | Gate | Status |
|------------|--------|----------|------|--------|
| `<src/ ou frontend/>` | `__%` | `__%` | 80% | ✅/❌ |

## Lacunas (só arquivos abaixo de 80%, ou com trecho relevante não coberto)

Omita esta seção (ou deixe "nenhuma") quando todo arquivo tocado está ≥ 80% — não liste arquivo já
coberto só para ser exaustivo.

| Arquivo | Cobertura | Linhas não cobertas |
|---------|-----------|----------------------|
| `<caminho/arquivo>` | `__%` | `<ex.: 42-47, 63>` |

## Observações

`<qualquer nota curta que mude a leitura do número acima — ex.: "trecho não coberto é código morto
sinalizado no code review", "falha intermitente confirmada em 3 execuções">` — ou "nenhuma".
