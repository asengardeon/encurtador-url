# QA Report — <Nome da Feature>

> Autor: agente `qa-engineer`
> PRD: `specs/<slug>/prd.md` | TRD: `specs/<slug>/trd.md`
> Fatia validada nesta rodada: `<F-1, ou "única — feature não fatiada">`
> PR desta fatia: <link do Pull Request revisado>
> Data:

Este documento é editado in-place a cada fatia (nunca recriado do zero) — ver seção "Histórico de
aprovações por fatia" abaixo para o veredito de fatias anteriores já mergeadas. **Quem lê este
arquivo numa rodada seguinte lê a tabela de histórico inteira, a rodada vigente e a penúltima** —
rodadas mais antigas são referência sob demanda, não leitura obrigatória (`AGENT-PREAMBLE.md` do
pipeline, seção "Artefato in-place que acumulou rodadas"). Identifique a rodada que você acrescenta
pelo mesmo identificador da tabela (`F-N` ou `hotfix-<data>`), para que a vigente seja localizável
sem varrer o arquivo.

## 1. Veredito geral (fatia desta rodada)

**Aprovado / Reprovado**

### Histórico de aprovações por fatia

Uma linha por rodada de QA (uma por fatia) — nunca sobrescreva o veredito de uma fatia já
aprovada e mergeada, acrescente uma linha nova.

| Fatia | PR      | Veredito | Data |
|-------|---------|----------|------|
| F-1   | `<link>` | `<veredito>` | `<data>` |

## 2. Cobertura de testes

| Métrica            | Resultado | Gate  | Status |
|---------------------|-----------|-------|--------|
| Cobertura de linhas  | __%       | 80%   | ✅/❌ |
| Cobertura de branches| __%       | 80%   | ✅/❌ |

Fonte: `<reaproveitado de specs/<slug>/coverage/<fatia>-<trilha>.md, commit <sha> | re-executado nesta rodada — arquivo estava ausente/desatualizado>`
Comando usado: `<comando de teste com cobertura>`

## 3. Critérios de aceite (PRD) cobertos por esta fatia

Só os critérios da fatia desta rodada (seção "Ordem de valor" do PRD / coluna "Fatia (PRD)" do
TRD) — critérios de fatias futuras ainda não implementadas não entram aqui.

| ID    | Critério                         | Teste automatizado         | Veredito |
|-------|------------------------------------|------------------------------|----------|
| US-1  | <cenário Gherkin resumido>          | `tests/.../test_x.py::...`   | ✅/❌   |

## 4. Regressão

Resultado da suíte completa (não só os testes novos): `<passou/falhou, X testes, Y falhas>`.

### 4b. Build/empacotamento real (se a fatia tem trilha de frontend ou gera artefato próprio)

Resultado do comando de build/empacotamento real de produção (`docs/STACK.md`), reaproveitado de
`specs/<slug>/coverage/<fatia>-<trilha>.md` ou reexecutado nesta rodada — `TESTING.md` do pipeline,
seção "Build/empacotamento real como parte da suíte completa". Lint/tipo/teste unitário sozinhos não
contam como suíte completa quando este campo se aplica. "Não aplicável" se a fatia é backend-only
sem etapa de empacotamento própria além dos testes.

## 5. Aderência a ports & adapters

Testes de domínio/aplicação rodam isolados de infraestrutura real? Sim/Não — se não, detalhe
onde a fronteira foi violada.

## 6. Achados (acumulado entre fatias — nunca recriado do zero)

Uma linha por achado, nunca removida — ao corrigir, atualize o `Status` da própria linha em vez
de apagá-la ou duplicá-la. Achado com `Status = corrigido` cuja `Fatia` já foi mergeada é
referência sob demanda, não leitura obrigatória nas rodadas seguintes (`docs/AGENT-PREAMBLE.md`,
seção "Artefato in-place que acumulou rodadas"). Achado `aberto` continua leitura obrigatória todo
round, mesmo que de uma fatia anterior.

| # | Fatia | Arquivo/cenário | Esperado | Observado | Severidade | Status |
|---|-------|-------------------|----------|-----------|------------|--------|

## 7. Pendências de validação (VALIDAR DEPOIS)

Critério de aceite ambíguo o suficiente para não dar veredito sozinho vira pergunta ao usuário;
se ele não souber responder agora, registre aqui em vez de decidir por conta própria. **ID no
formato `QA-<AAAA-MM-DD>-<slug-curto>`** (data + slug curto do próprio item — nunca um contador
sequencial simples como `QA-1`/`QA-2`): duas branches de fatia paralelas calculando o "próximo
número" a partir da própria cópia local já geraram colisão real de ID em merge (mesmo critério de
`docs/LESSONS-LEARNED.md`, `gates/licoes.md` do pipeline). Na rara colisão de duas entradas com data e
slug idênticos, acrescente um sufixo numérico ao segundo (`-2`, `-3`, ...) no momento do merge.

| ID                              | Pergunta                          | Contexto                              | Status              |
|----------------------------------|--------------------------------------|-------------------------------------------|------------------------|
| QA-2026-08-25-criterio-timeout   | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>            | pendente / validado |

## 8. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 9. Próximo passo

`/sdd-security` (se aprovado, para esta fatia) ou `/sdd-implement` (se reprovado, com os achados
acima).
