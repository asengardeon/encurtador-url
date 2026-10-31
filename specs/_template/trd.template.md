# TRD — <Nome da Feature>

> Status: rascunho | em revisão | aprovado
> Autor: agente `architect`
> PRD relacionado: `specs/<slug>/prd.md`
> ADRs relacionados: (link, se houver)

## 1. Contexto

Resumo de uma frase do problema (ver PRD) e a abordagem técnica escolhida.

## 2. Stack Tecnológica

Decidida **antes** de qualquer coisa abaixo que dependa dela (nunca implícita dentro de
Ports/Adapters/Modelo de dados). Ver processo de decisão em `.claude/agents/architect.md`.

- **Linguagem/runtime**: <...>
- **Framework principal**: <... ou "nenhum">
- **Persistência**: <... ou "nenhuma">
- **Gerenciador de pacotes**: <...>
- **Proveniência desta decisão**: `<reaproveitada de docs/STACK.md deste projeto / reaproveitada
  do harness global ~/.claude/stack-defaults.md, confirmada com o usuário / decidida nesta sessão
  com o usuário>`

Se esta é a primeira vez que a stack foi decidida neste projeto, `docs/STACK.md` foi
criado/atualizado com o mesmo conteúdo — próximas features reaproveitam a partir de lá.

## 3. Visão de arquitetura

Como a feature se encaixa em ports & adapters (`ARCHITECTURE.md` do pipeline):

```
[adapter de entrada] → [caso de uso] → [port] ← [adapter de saída]
                              ↓
                        [domínio]
```

Descreva os componentes reais desta feature nesse diagrama. Se a feature inclui frontend, indique
também onde ele entra (consumindo o adapter de entrada via o contrato da seção 8).

## 4. Modelo de domínio

Entidades, value objects, invariantes e regras de negócio — sem menção a framework/infra.

## 5. Ports (contratos)

Para cada port novo/alterado:

### `<NomeDoPort>`

- Responsabilidade única: <...>
- Métodos: `<assinatura>` → <retorno/erros esperados>
- Quem implementa (adapter de saída): <...>
- Quem consome (caso de uso): <...>

## 6. Casos de uso

Mapeamento explícito de critério de aceite (PRD) → caso de uso:

| Critério de aceite (PRD) | Caso de uso                | Ports usados |
|---------------------------|-----------------------------|--------------|
| US-1 cenário X             | `<NomeDoCasoDeUso>`         | `<Port>`     |

## 7. Adapters

### Entrada

- `<AdapterDeEntrada>`: responsabilidade, como aciona o(s) caso(s) de uso. Implementado pelo
  `backend-developer`.

### Saída

- `<AdapterDeSaida>`: responsabilidade, qual port implementa, dependências externas reais.
  Implementado pelo `backend-developer`.

## 8. Contrato Frontend↔Backend (API)

Preencher só se a feature inclui frontend. Isso é o que permite `backend-developer` e
`frontend-developer` desenvolverem em paralelo sem esperar um pelo outro — cada um implementa
contra este contrato, não contra a implementação real do outro lado.

- **Aplicável?** `<sim / não aplicável, porque é uma feature só de backend ou só de frontend>`
- **Endpoints/mensagens**: `<método + rota, ou nome do evento/mensagem>`
- **Request**: `<schema do payload de entrada>`
- **Response**: `<schema do payload de saída, incluindo casos de sucesso>`
- **Formato de erro padrão**: `<estrutura de erro, códigos usados>`
- **Autenticação/autorização**: `<mecanismo, se houver, ou "nenhuma">`
- **ADR relacionado**: `<link, se este contrato estabelece uma convenção reutilizável por todo o
  app — senão "nenhum, decisão local desta feature">`

## 9. Modelo de dados / contratos externos

Schema, payloads de API, formato de eventos — o que for aplicável (além do contrato da seção 8,
se houver).

## 10. Pilares de engenharia de software

Cada pilar exige resposta explícita — "não se aplica, porque X" é uma resposta válida; em branco
não é. Detalhe conceitual de cada pilar em `ENGINEERING-PILLARS.md` do pipeline.

- **Performance**: <latência/throughput esperado, ou "não se aplica, porque..."`>
- **Escalabilidade**: <a feature introduz estado em memória do processo? aguenta múltiplas
  instâncias? ou "não se aplica, porque..."`>
- **Resiliência**: <para cada dependência externa nova: timeout/retry/circuit breaker/fallback
  definidos, ou "não se aplica, porque..."`>
- **Disponibilidade**: <impacto se esta feature ficar indisponível — crítico/degradação
  aceitável/irrelevante — e por quê`>
- **Observabilidade** (logs/métricas mínimas): <o que precisa ser logado/medido nas fronteiras de
  adapter para diagnosticar problema em produção`>
- **Manutenibilidade**: <desvio deliberado de SOLID/Clean Code (`ARCHITECTURE.md` do pipeline), se
  houver, com justificativa; senão "nenhum desvio"`>
- **Impacto em infraestrutura para o SRE revisar:** <fila? cache? novo serviço? escalonamento?
  ou "nenhum">
- **Indicadores técnicos herdados do PRD** (seção "Indicadores técnicos a observar" do
  `prd.md`): para cada um (volumetria, segurança, legal), a decisão tomada aqui ou a justificativa
  de por que foi adiada — nunca ignorado silenciosamente. O indicador de segurança é aprofundado
  depois pelo agente `security-engineer` (`/sdd-security`); aqui só a decisão de design de alto
  nível que o afeta (ex.: precisa de autenticação? qual dado é sensível?).

## 11. Plano de testes (alto nível)

- Unitário: quais componentes de domínio/aplicação, com quais dublês de port.
- Integração: quais adapters, contra o quê (ex.: banco em memória/container de teste).
- E2E: quais fluxos ponta a ponta.
- Frontend (se aplicável): componentes/serviços testados com dublê do contrato da seção 8.
- Meta de cobertura: 80% por pacote (`src/` e, se aplicável, `frontend/` — ver `TESTING.md` do pipeline).

## 12. Riscos e trade-offs

Riscos técnicos identificados e a decisão tomada (com justificativa).

**Toda afirmação sobre comportamento concreto de biblioteca de terceiro ou de runtime** — tipo de
exceção lançada, limite numérico, formato gravado, o que um `catch` captura — vem marcada como
**`[medido]`** (com a versão da biblioteca e como foi medido) ou **`[não medido]`** (hipótese vinda
de leitura de documentação/código, a verificar por quem depender dela). Afirmações desse tipo
*parecem* factuais e não convidam à verificação; a marcação é o que transforma uma armadilha
silenciosa em item acionável, a custo praticamente zero para quem escreve.

O mesmo vale para a decisão registrada ao lado do risco: dizer que algo "vira falha daquela linha"
(ou qualquer outra alegação de **contenção**) é uma afirmação sobre o próprio código — **cite o
ponto onde a contenção acontece**. Se ninguém conseguir citar o ponto, a contenção provavelmente
não existe.

| Risco | Impacto | Decisão / mitigação | Evidência |
|-------|---------|---------------------|-----------|
| <risco> | <impacto real, não o suposto> | <decisão, citando o ponto de contenção quando alegar contenção> | `[medido]` (biblioteca X vN, como) ou `[não medido]` |

## 13. Decomposição de tarefas e dependências (fatias verticais de entrega)

Ponto de partida: a seção "Ordem de valor / dependências entre histórias (fatias verticais de
entrega)" do PRD. Aqui a decomposição é técnica, por tarefa, incluindo dependências que só a
arquitetura revela — mas **agrupe as tarefas por fatia de forma que, ao final de cada fatia, o
resultado continue demonstrável de ponta a ponta** (quando full-stack: backend e frontend da
mesma fatia concluídos juntos, nunca "todo o backend primeiro, todo o frontend depois"). É isso
que permite entregar e mostrar a spec completa aos poucos, em vez de só no fim.

**Tarefa de investigação vem primeiro.** Se alguma decisão desta spec depende da forma concreta de um
contrato com um sistema externo que ninguém observou (seletor de DOM de terceiro, formato de resposta
de API não documentada, layout de arquivo de parceiro, esquema de webhook, nome de campo de SSO), a
tarefa que captura essa estrutura real é a **primeira** tarefa da primeira fatia que depende dela —
antes de qualquer tarefa que escreva código de produção contra ela, e com Issue GitHub própria como
qualquer outra. Ela entrega um artefato em `specs/<slug>/` com a estrutura capturada (snapshot
ARIA/DOM, payload real, amostra do arquivo) e uma seção "correções obrigatórias ao desenho do TRD";
este TRD é emendado antes de a fatia prosseguir. O motivo é que o dublê escrito a partir de uma
estrutura suposta **valida a suposição, não a realidade**: a suíte fica verde e nenhuma etapa de
revisão posterior consegue distinguir isso de uma suíte verde contra a realidade.

**...exceto quando ela mede a plataforma contra o artefato desta própria fatia.** Uma investigação
que mede a plataforma **isoladamente** ("que código HTTP o serviço devolve para endereço revogado")
roda antes do código e pode ter "Depende de" = `nenhuma`. Uma que mede o comportamento da
plataforma **contra artefato produzido por outras tarefas desta fatia** ("o navegador oferece
instalar com *este* manifest e *este* service worker?") não pode precedê-lo: a coluna "Depende de"
lista as tarefas cujo artefato a medição exige, o texto da tarefa diz que ela roda **ao fim** da
fatia, e a seção 11 (plano de testes) não afirma que as investigações rodam antes do código sem
qualificar quais. Teste de um passo: *a medição é possível com o repositório como está hoje?*

| ID   | Tarefa                    | Trilha                    | Fatia (PRD) | Depende de | Validação na fatia | Status | Issue GitHub |
|------|------------------------------|------------------------------|--------------|---------------|--------------------|-----------|------------------|
| T-1  | <descrição da tarefa>         | backend / frontend / ambos    | F-1           | nenhuma        | — / segurança / ux / sre | pendente  | `<#N ou "não espelhada">` |

A coluna **Validação na fatia** decide se uma das três etapas de fim de spec (4b UX, 6 segurança,
7 SRE) é **antecipada** para a fatia desta tarefa, em vez de rodar sobre o diff acumulado no fim da
spec (`docs/SDD-WORKFLOW.md`, seção "Cadência: o que roda por fatia e o que roda ao final da
spec"). `—` é o caso normal. **Em branco não é opção**, e a marcação não é julgamento quando a
tarefa toca:

| Marque | Quando a tarefa toca |
|--------|------------------------|
| `segurança` | autenticação, autorização, gestão de sessão, dado pessoal/sensível, ou ponto de entrada que aceita identificador externo |
| `sre` | `ci.yml`/`cd.yml`, `Dockerfile`/compose, `infra/terraform`, segredo ou variável de ambiente de deploy, ou campo que passa a ser obrigatório numa rota ativa já consumida por cliente implantado |
| `ux` | fluxo ou tela **nova** exposta ao usuário final (ajuste dentro de fluxo que a 4b já revisou não qualifica) |

Fora desses casos a antecipação é julgamento do `architect` — permitida, com o motivo em uma linha
na descrição da tarefa — ou pedido explícito do usuário. Valores combinam quando mais de uma se
aplica (ex.: `segurança, sre`).

### Janelas de quebra de contrato entre fatias

Preencher só se alguma fatia acima introduz uma mudança de contrato obrigatória (campo novo
obrigatório numa API, mensagem/evento incompatível, remoção de suporte a formato antigo) que só
fica coerente depois que outra fatia posterior (ex.: a que atualiza o cliente/frontend) também
mergear — nunca deixar essa janela implícita. Se nenhuma fatia introduz esse tipo de janela,
escreva "não aplicável".

| Fatia que quebra | O que quebra | Só fica coerente depois de | Mitigação decidida | `!` no título do PR |
|--------------------|-----------------|--------------------------------|------------------------|------------------|
| F-1                  | <ex.: campo `tenantSlug` passa a ser obrigatório em `POST /auth/login`> | F-3 (seletor de tenant na UI) | <ex.: campo aceito como opcional com fallback até F-3 mergear / segurar merge de F-1 até F-3 estar pronta / janela aceita conscientemente, com justificativa> | sim / não |

A última coluna é acionável, não informativa: em repositório com release automatizado por
Conventional Commits, o `!` antes dos dois-pontos no título do PR (`feat(0005)!: ...`) é o **único**
sinal de quebra que sobrevive ao squash merge — o subject do commit de squash é o título do PR. Sem
ele, uma mudança que rejeita tudo que o usuário tinha em mãos é publicada como MINOR, e o número de
versão é o único aviso que chega a quem só baixa o artefato da página de Releases. Marque "sim"
sempre que a fatia carregar a quebra, mesmo que a mitigação escolhida reduza o impacto; "não" só
quando a fatia não é a portadora da quebra.

A coluna **Status** é a fonte de verdade de onde cada tarefa está, mantida **in-place** por quem
causa cada transição — nunca inferida depois por outra etapa. `architect` inicializa toda tarefa
nova como `pendente`. Ciclo de vida completo e responsabilidade de cada transição em
`gates/status-de-tarefas.md` do pipeline: `pendente` → `em andamento` → `implementado`
→ `aprovado` → `concluído (mergeado)`, com `bloqueado` como estado de exceção (uma revisão
reprovou; volta a `em andamento` quando a correção começa).

**Toda tarefa desta tabela precisa de uma Issue GitHub associada antes do TRD ser considerado
pronto para aprovação — obrigatório, não mais uma preferência.** O `architect` verifica remote
GitHub configurado e autenticado ao preencher esta tabela: se houver, cria as issues via `gh issue
create` (informando o usuário, sem perguntar se ele quer); se não houver, o TRD não pode ser
finalizado até o usuário configurar `git remote` + `gh auth login` — diferente do PRD, que não
exige GitHub, a decomposição de tarefas exige. Nenhuma fatia desta spec pode ser iniciada por
`/sdd-implement` sem que todas as suas tarefas tenham a coluna "Issue GitHub" preenchida.

Toda issue de tarefa criada aqui é identificável contra a spec e o tipo de trabalho que
representa:
- **Milestone**: um milestone por spec, nomeado com o `<slug>` da feature (ex.:
  `0012-excluir-eventos-cancelados`) — cria o milestone no repositório do projeto se ainda não
  existir (`gh api repos/<owner>/<repo>/milestones -f title="<slug>" -f state="open"`, verificando
  antes com `gh api repos/<owner>/<repo>/milestones --jq '.[].title'` para não duplicar) e associa
  toda issue de tarefa desta spec a ele (`gh issue create --milestone "<slug>" ...`). Agrupa todas
  as tarefas da mesma spec com acompanhamento nativo de % concluído no GitHub.
- **Label de tipo**: `enhancement` para a imensa maioria das tarefas (implementação nova, o caso
  padrão de uma tarefa de TRD), `bug` só se a tarefa for uma correção de defeito já existente, e
  `documentation` se a tarefa for só de documentação — todos labels padrão do GitHub, presentes na
  maioria dos repositórios; confirme que existem (`gh label list --repo <owner>/<repo> --json
  name`) e crie se faltar antes de usar.
- **Label de trilha**: `backend`, `frontend` ou `ambos`, espelhando a coluna **Trilha** da tabela
  acima — crie o label no repositório do projeto se ainda não existir (`gh label create`, mesma
  mecânica acima).

Ciclo de vida de cada issue criada: o PR da fatia que cobre a tarefa referencia `Closes #N`
(`backend-developer`/`frontend-developer`, ao abrir o PR); `sre`, ao aprovar a fatia, comenta na
issue documentando o que foi implementado (link do PR e dos artefatos de revisão); a issue fecha
sozinha quando o usuário mergear o PR — nunca fechada manualmente antes disso. Sempre que a coluna
Status de uma tarefa muda, o mesmo agente/skill responsável por essa transição comenta a mudança
na issue associada (`gh issue comment`), para o TRD e o GitHub nunca divergirem.

## 14. Controle de versão (GitHub Flow, por fatia)

Cada fatia vertical da seção 13 é entregue como sua **própria branch/PR**, incremental sobre a
fatia anterior já mergeada em `main` — nunca uma branch/PR única cobrindo todas as fatias de uma
vez (ver `GIT-WORKFLOW.md` do pipeline). A branch da fatia N só é criada depois que o PR da fatia N-1
está mergeado.

| Fatia | Branch                                              | PR                            | Status                              |
|-------|--------------------------------------------------------|----------------------------------|------------------------------------------|
| F-1   | `feature/<NNNN-slug>` (ou `feature/<NNNN-slug>/f-1` se houver mais de uma fatia) | <link, preenchido quando existir> | aberto / mergeado |

- Se a fatia é full-stack: dentro da branch dessa fatia, `backend-developer` trabalha em
  `src/`+`tests/`, `frontend-developer` em `frontend/`, ambos na mesma branch — árvores de
  diretório separadas evitam conflito de merge.
- Se houver só uma fatia (feature pequena, não fatiada), a tabela tem uma única linha e a branch
  usa o nome simples `feature/<NNNN-slug>`, sem sufixo de fatia.

## 15. Pendências de validação (VALIDAR DEPOIS)

| ID     | Pergunta                          | Contexto                              | Status              |
|--------|--------------------------------------|-------------------------------------------|------------------------|
| TRD-1  | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>            | pendente / validado |

## 16. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 17. Aprovação

- [ ] Aprovado por: <usuário> em <data>
