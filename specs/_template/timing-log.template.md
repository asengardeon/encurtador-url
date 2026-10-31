# Log de tempo de execução — <slug>

Registro append-only do tempo que cada etapa desta spec levou — invocação de agente **e** trabalho
conduzido pelo próprio orquestrador (ver "Trabalho conduzido pelo orquestrador" abaixo) —, preenchido pelo
**orquestrador** de cada skill (`/sdd-prd`, `/sdd-trd`, `/sdd-implement`, `/sdd-code-review`,
`/sdd-qa`, `/sdd-security`, `/sdd-sre`), nunca pelos agentes em si (eles não têm visibilidade do
próprio horário de início/fim do ponto de vista de quem os invocou). Uma linha por invocação —
nunca sobrescreva uma linha já registrada, mesmo numa retomada/correção.

Consumido pela retrospectiva de fatia do `sre` (`.claude/agents/sre.md`, seção "Retrospectiva da
fatia") para identificar etapas anormalmente lentas ou padrões de custo de tempo entre fatias.

## O que a coluna "Duração" mede

**Tempo de trabalho do(s) agente(s) da invocação — não wall-clock bruto do início ao fim.** A
diferença aparece sempre que a invocação inclui uma pausa que não é trabalho de agente nenhum, e o
caso mais comum é a **espera por aprovação humana**: um agente que devolve o plano da Fase 1 e fica
parado até o usuário responder (`/sdd-implement`, passo 5b) pode acumular horas de wall-clock sem
nenhum trabalho acontecendo. Nesse caso, registre a **soma das durações reais das invocações de
subagente envolvidas** (o `duration_ms` que cada notificação de conclusão de subagente já reporta),
e acrescente uma nota parentética explícita dizendo o que ficou de fora:

```
| 2026-09-22T03:47:59Z | Implementação | backend-developer | F-3 | padrão da sessão | 27m58s (tempo de trabalho do agente, exclui a espera de aprovação do usuário entre Fase 1 e Fase 2 — gap real de wall-clock foi maior) |
```

Sem essa distinção, a linha registra o tempo de resposta do usuário como se fosse custo do
pipeline, e a retrospectiva que lê este arquivo conclui que a etapa foi anormalmente lenta quando
não foi. A nota parentética não é opcional quando a exclusão acontece: quem ler o log depois
precisa entender a diferença sem adivinhar. Já aconteceu de verdade: um gap de aprovação de mais de
10h teria virado uma linha "Implementação | 10h+" se a instrução tivesse sido seguida como
wall-clock puro.

O inverso também vale: **nunca "limpe" um número medido**. Se a duração foi de fato dominada por
trabalho lento (ou por uma espera que o próprio agente escolheu fazer), registre o valor medido e
explique a causa na nota — inventar um número arredondado corrompe a série histórica tanto quanto
deixar o outlier sem explicação.

**Agente retomado via `SendMessage`: confira o `duration_ms` contra o relógio de parede antes de
registrá-lo.** Em agente retomado, o `duration_ms` da notificação de conclusão **pode** ser
cumulativo desde a criação do agente, e não o da invocação isolada — e o comportamento observado é
inconsistente, tanto entre campos (`tool_uses` e `subagent_tokens` tendem a crescer entre rodadas)
quanto entre invocações (num caso real, três rodadas do mesmo agente reportaram 48m → 83m → 12m,
com a segunda não cabendo no wall-clock disponível depois da primeira; noutra spec, três rodadas do
mesmo agente tiveram durações decrescentes, incompatíveis com cumulativo). Se o número não couber
depois da rodada anterior, registre a diferença que o relógio sustenta e **diga na própria linha**
que o valor reportado é ambíguo e não deve ser somado ingenuamente às outras rodadas daquele
agente:

```
| 2026-10-03T20:00:00Z | Correção pontual | frontend-developer | F-1 | padrão da sessão | ~35m (duration_ms reportado de 83m04s é incompatível com o relógio — provavelmente cumulativo desde a criação do agente; não somar com a linha da rodada anterior) |
```

**A assinatura de detecção é a monotonia entre rodadas — e, detectada, o número a registrar é a
diferença.** Contador cumulativo e contador por invocação se distinguem sem precisar do relógio:
compare o mesmo campo entre rodadas sucessivas do mesmo agente. Se ele **cresce monotonicamente**
enquanto o trabalho de cada rodada oscila, é total de vida, não custo da rodada. Já aconteceu de
verdade: os `subagent_tokens` de um `frontend-developer` ao longo de 8 invocações foram
`288.900 → 380.204 → 437.478 → 527.579 → 621.760 → 648.938 → 719.348 → 761.011` — monotônicos —
com os usos de ferramenta oscilando (150, 56, 30, 52, 66, 21, 42, 31). Nesse caso, **registre a
diferença entre o contador desta notificação e o da notificação anterior do mesmo agente**
(duração e tokens), não o valor bruto, e diga na linha que é diferença:

```
| 2026-10-06T14:12:40Z | Correção pontual | frontend-developer | F-2 | padrão da sessão | 11m18s (diferença contra a rodada anterior do mesmo agente; contadores reportados são cumulativos desde a criação — 719.348 → 761.011 tokens = 41.663 nesta rodada) |
```

Sem isso, a conclusão se inverte: 761k lido como custo de uma rodada faz o log afirmar que
**retomar agente é caro**, quando a medida correta prova o contrário — nessa mesma fatia, o custo
marginal das 7 rodadas de correção foi 472.111 tokens (~67k/rodada), enquanto cada instância fresca
de revisor custou 223k–266k só para se orientar.

**Anotar o horário antes do `SendMessage` resolve o caso na origem** e é a instrução que vale
seguir — `/btt-sdd:implement`, seção "Retomando para corrigir achados de revisão", passo 1.
A conferência acima é a rede de segurança para quando esse horário não existe, e não é zelo
opcional: a retrospectiva de fatia (`/btt-sdd:sre`, passo 5c) lê este arquivo como evidência
concreta de performance, e uma linha de 83m que na verdade foram ~35m faz a análise concluir o
oposto do que os dados mostram.

## Uma linha por etapa, sempre

**Etapa com trabalho real e nenhuma linha é pior que uma linha imperfeita.** Toda etapa desta fatia
aparece no log, inclusive quando o trabalho de orquestrador nela foi zero — aí a linha registra
`0m` com a justificativa, em vez de não existir. A ausência de linha é indistinguível de "não
medido", e a série passa a comparar coisas diferentes: já aconteceu de o SRE de uma fatia ter só a
linha do agente (47m32s) enquanto a rodada roteava 6 pendências ao orquestrador, entregava um texto
de issue para ele criar, e conduzia a retrospectiva da fatia — contra a fatia anterior, que tinha a
linha equivalente de ~15m. A série comparou "SRE F-1 = 29m27s + 15m" com "SRE F-2 = 47m32s + 0".

**Nenhuma linha nomeia duas etapas na coluna `Etapa`.** Quando duas etapas rodam em paralelo — o
que é legítimo —, são **duas linhas com o mesmo horário de início**, exatamente como
`/btt-sdd:implement`, passo 4d, já manda fazer para as duas trilhas de uma feature full-stack. Uma
linha "Segurança + code review" deixa uma das duas etapas com zero minuto atribuível: já aconteceu
de a segurança de uma fatia ficar sem nenhum tempo de orquestrador registrado, apesar de 2
perguntas de decisão intermediadas e de uma convenção gravada em `docs/PROJECT-CONVENTIONS.md`.

**Por que os três defeitos importam juntos:** contador cumulativo, etapa sem linha e linha agregada
empurram todos na **mesma direção** — superestimam as etapas de agente e subestimam as do
orquestrador, que é justamente o viés que a seção "Trabalho conduzido pelo orquestrador" existe para
corrigir. Numa fatia medida, 29% do esforço era do orquestrador (~177 min contra ~439 de agente): o
viés não é marginal.

## Trabalho conduzido pelo orquestrador (sem agente)

**`orquestrador (sem agente)` é um valor legítimo da coluna `Agente`** — não uma convenção
improvisada na hora. Registre uma linha assim sempre que o trabalho da etapa foi conduzido por quem
orquestra, sem invocar agente nenhum:

- **Tarefa de investigação executada sem agente** — ex.: navegar um sistema externo ao vivo para
  capturar a estrutura real (DOM, payload, layout de arquivo) que o TRD supôs.
- **Rodadas de `AskUserQuestion`** que custaram tempo real de relógio entre duas invocações —
  coleta de decisões de produto ou técnicas, não a espera passiva por uma aprovação (essa é o caso
  da seção acima).
- **Intermediação de plano/pergunta de subagente isolado** — apresentar ao usuário o que o agente
  não conseguiu perguntar e retomá-lo (`/btt-sdd:implement`, passos 4c e 5b; `/btt-sdd:sre`, passo 4b).

```
| 2026-09-25T14:02:11Z | Implementação | orquestrador (sem agente) | F-1 | padrão da sessão | 14m09s (investigação ao vivo do DOM real — tarefa T-1; derrubou o seletor desenhado no TRD) |
```

**Por que isso não é burocracia:** a retrospectiva de fatia lê este arquivo para achar etapas
anormalmente lentas. Sem essas linhas, ela lê uma série que **subestima** sistematicamente as
etapas conduzidas pelo orquestrador e **superestima** as conduzidas por agente. Já aconteceu de uma
etapa de PRD com 13m30s de agente ter ~23 min de trabalho de orquestrador invisível no log — duas
rodadas de `AskUserQuestion` com 7 decisões de produto e uma investigação que **reescreveu a
motivação da spec**, ou seja, a atividade de maior valor da etapa, sem linha nenhuma.

## A coluna "Modelo": qual tier pagou por esta invocação

`padrão da sessão` ou `econômico` — os dois tiers de `MODEL-TIERING.md` do pipeline (o nome
concreto do modelo cabe entre parênteses, quando quem registra o souber). Toda linha preenche a
coluna, inclusive `orquestrador (sem agente)`, que roda no padrão da sessão por definição.

Só **sub-tarefa auxiliar de trabalho bem definido** aparece como `econômico`; nenhuma das etapas do
pipeline aparece assim, porque etapa escreve artefato aprovável e decide gate. Uma linha de etapa
marcada `econômico` é um gate violado, não uma economia — é exatamente o que esta coluna torna
auditável depois do fato.

**Sub-tarefa repromovida gera duas linhas, não uma corrigida** (o arquivo é append-only): a
econômica que não entregou e a do padrão da sessão que a substituiu, com a nota dizendo o que veio
inutilizável. É esse par que a retrospectiva de fatia lê para dizer se o rebaixamento se pagou —
uma linha sobrescrita apaga justamente a evidência de que não se pagou, e a mesma sub-tarefa é
rebaixada de novo na fatia seguinte.

```
| 2026-10-07T11:02:40Z | SRE | sub-tarefa (leitura do timing-log) | F-2 | econômico | 1m12s |
```

## Desfecho excepcional: marcador no início da nota, não prosa livre na coluna "Etapa"

Quando a invocação não terminou no resultado normal da etapa (recusou produzir veredito, repetiu
trabalho de uma rodada anterior) — não só "demorou mais ou menos" —, prefixe a nota da coluna
`Duração` com um marcador padronizado, em vez de descrever o desfecho em prosa livre dentro da
coluna `Etapa`. Dois marcadores cobrem os casos observados até agora:

- `[RECUSADO]` — o agente não produziu veredito (pré-condição não satisfeita, ordem errada,
  artefato de entrada ausente/reprovado). Diga o motivo no resto da nota.
- `[RETRABALHO]` — a invocação repetiu/corrigiu trabalho de uma rodada anterior da mesma etapa
  (achado de revisão, correção pontual).

```
| 2026-10-18T09:12:00Z | Segurança | security-engineer | F-3 | padrão da sessão | 2m14s ([RECUSADO] QA ainda não tinha rodado — invocado fora de ordem pelo orquestrador; nenhum veredito produzido) |
```

Isso mantém a coluna `Etapa` só com o nome da etapa e torna o desfecho excepcional localizável por
busca textual (`grep '\[RECUSADO\]'`) em vez de depender de reconhecer prosa livre — baixo custo,
é a mesma nota parentética que a coluna `Duração` já carrega para outras explicações, só com um
prefixo padronizado quando o desfecho, não só o tempo, foi anormal.

| Início (UTC)          | Etapa           | Agente               | Fatia | Modelo | Duração |
|------------------------|-----------------|-----------------------|-------|--------|---------|
| <AAAA-MM-DDThh:mm:ssZ> | <PRD/TRD/Implementação/Code review/QA/Segurança/SRE> | <nome do agente, ou `orquestrador (sem agente)`> | <F-N ou —> | <padrão da sessão / econômico> | <XmYs> |
