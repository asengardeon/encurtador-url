# Security Review — <Nome da Feature>

> Autor: agente `security-engineer`
> QA report: `specs/<slug>/qa-report.md`
> Fatia revisada nesta rodada: `<F-1, ou "única — feature não fatiada">`
> PR desta fatia: <link do Pull Request>
> Data:

Este documento é editado in-place a cada fatia (nunca recriado do zero) — ver seção "Histórico de
aprovações por fatia" abaixo para o veredito de fatias anteriores já mergeadas. **Quem lê este
arquivo numa rodada seguinte lê a tabela de histórico inteira, a rodada vigente e a penúltima** —
rodadas mais antigas são referência sob demanda, não leitura obrigatória (`AGENT-PREAMBLE.md` do
pipeline, seção "Artefato in-place que acumulou rodadas"). Identifique a rodada que você acrescenta
pelo mesmo identificador da tabela (`F-N` ou `hotfix-<data>`), para que a vigente seja localizável
sem varrer o arquivo.

## 1. Veredito geral (fatia desta rodada)

**Aprovado / Aprovado com ressalvas / Reprovado**

### Histórico de aprovações por fatia

Uma linha por rodada de revisão (uma por fatia) — nunca sobrescreva o veredito de uma fatia já
aprovada e mergeada, acrescente uma linha nova.

| Fatia | PR      | Veredito | Profundidade | Commit | Data |
|-------|---------|----------|--------------|--------|------|
| F-1   | `<link>` | `<veredito>` | `<completo\|fast-path>` | `<sha>` | `<data>` |

`Profundidade`: `completo` (todas as áreas revisadas de verdade) ou `fast-path` (áreas sem
superfície tocada foram condensadas — critério em `.claude/agents/security-engineer.md`). Na
fatia final que fecha o spec, é sempre `completo`. `Commit`: SHA do HEAD no momento desta revisão
— usado para calcular o diff acumulado na auditoria final.

## 2. Superfície de ataque (resumo)

Fronteiras de confiança desta feature (entrada de usuário, chamada externa, arquivo, variável de
ambiente) e por que cada uma é ou não uma superfície de risco.

## 3. OWASP Top 10 (aplicado ao que existe)

| Categoria                                 | Aplicável? | Avaliação                              |
|---------------------------------------------|--------------|-------------------------------------------|
| Injeção                                      |              |                                             |
| Quebra de autenticação                        |              |                                             |
| Exposição de dados sensíveis                  |              |                                             |
| Controle de acesso quebrado                    |              |                                             |
| Configuração insegura                          |              |                                             |
| Componentes com vulnerabilidade conhecida       |              |                                             |
| Falhas de log/monitoramento de segurança         |              |                                             |

(Toda linha precisa de avaliação — "não aplicável" com justificativa é uma resposta válida, em
branco não é.)

## 4. Gestão de segredos

Segredos hardcoded em código/config/log? Segredos via variável de ambiente/secret manager?

## 5. Autenticação / autorização

Se a feature tem noção de identidade/permissão: onde é verificada, e se todo caminho relevante
passa por essa verificação. Se não se aplica, diga por quê.

## 6. Validação de entrada

Toda fronteira de confiança (CLI, request, evento) valida antes de usar? Liste as fronteiras e o
que valida cada uma.

## 7. Dependências

Pacotes novos/alterados no manifesto de dependências, e qualquer vulnerabilidade conhecida
identificável com as ferramentas disponíveis. Se não há scanner automatizado no CI, recomende
(sem implementar aqui).

## 8. Achados (acumulado entre fatias — nunca recriado do zero)

Uma linha por achado, nunca removida — ao corrigir, atualize o `Status` da própria linha em vez
de apagá-la ou duplicá-la. Achado com `Status = corrigido` cuja `Fatia` já foi mergeada é
referência sob demanda, não leitura obrigatória nas rodadas seguintes (`docs/AGENT-PREAMBLE.md`,
seção "Artefato in-place que acumulou rodadas"). Achado `aberto` continua leitura obrigatória todo
round, mesmo que de uma fatia anterior.

| # | Fatia | Área | Esperado | Observado | Severidade | Status |
|---|-------|------|----------|-----------|------------|--------|

## 9. Pendências de validação (VALIDAR DEPOIS)

**ID no formato `SEC-<AAAA-MM-DD>-<slug-curto>`** (data + slug curto do próprio item — nunca um
contador sequencial simples como `SEC-1`/`SEC-2`): duas branches de fatia paralelas calculando o
"próximo número" a partir da própria cópia local já geraram colisão real de ID em merge (mesmo
critério de `docs/LESSONS-LEARNED.md`, `gates/licoes.md` do pipeline). Na rara colisão de duas entradas
com data e slug idênticos, acrescente um sufixo numérico ao segundo (`-2`, `-3`, ...) no momento
do merge.

| ID                              | Pergunta                          | Contexto                              | Status              |
|----------------------------------|--------------------------------------|-------------------------------------------|------------------------|
| SEC-2026-08-25-criterio-timeout  | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>            | pendente / validado |

## 10. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 11. Próximo passo

`/sdd-sre` (se aprovado, para esta fatia) ou `/sdd-implement` (se reprovado, com os achados
acima).
