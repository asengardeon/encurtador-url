# SRE Review — <Nome da Feature>

> Autor: agente `sre`
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
sem varrer o arquivo. Boa parte do checklist abaixo tende a se repetir sem mudança entre fatias da
mesma feature — quando não houver mudança relevante desde a última fatia aprovada, diga isso
explicitamente em vez de preencher evidência idêntica sem necessidade.

## 1. Veredito geral (fatia desta rodada)

**Aprovado / Aprovado com ressalvas / Reprovado**

### Histórico de aprovações por fatia

Uma linha por rodada de revisão (uma por fatia) — nunca sobrescreva o veredito de uma fatia já
aprovada e mergeada, acrescente uma linha nova.

| Fatia | PR      | Veredito | Profundidade | Commit | Data |
|-------|---------|----------|--------------|--------|------|
| F-1   | `<link>` | `<veredito>` | `<completo\|fast-path>` | `<sha>` | `<data>` |

`Profundidade`: `completo` (checklist revisado de verdade) ou `fast-path` (áreas sem superfície
tocada foram condensadas — critério em `.claude/agents/sre.md`). Na fatia final que fecha o spec,
é sempre `completo`. `Commit`: SHA do HEAD no momento desta revisão — usado para calcular o diff
acumulado na auditoria final.

## 2. CI

- [ ] Lint roda no pipeline
- [ ] Testes rodam no pipeline
- [ ] Gate de cobertura 80% aplicado e falha o build se violado
- [ ] Pipeline falha rápido e com mensagem clara

Evidência/observações:

## 3. CD

- [ ] Deploy só roda após CI verde
- [ ] `terraform apply` gated (plan revisável, aprovação manual quando aplicável)
- [ ] Estratégia de rollback documentada

### Proteção de `main` (GitHub Flow, ver `GIT-WORKFLOW.md` do pipeline)

Uma linha por sub-condição de propósito: um item único agregando as seis esconde a que falha atrás
de um ✅ (`gates/governanca.md` do pipeline). Sub-condição que falha vira achado, não observação na
evidência.

- [ ] Push direto em `main` bloqueado
- [ ] PR obrigatório antes do merge
- [ ] Status checks do CI obrigatórios
- [ ] Checks obrigatórios em modo `strict` (branch atualizada com `main` antes do merge)
- [ ] Force-push e deleção de `main` bloqueados
- [ ] Proteção aplicada também a administradores (`enforce_admins`) — a única camada aqui que não
  depende de um agente obedecer

Evidência/observações:

## 4. Docker

- [ ] Build multi-stage, imagem final mínima
- [ ] Processo roda como usuário não-root
- [ ] Sem segredo hardcoded na imagem
- [ ] Healthcheck definido (se aplicável)

Evidência/observações:

## 5. Terraform

- [ ] Estado remoto configurado
- [ ] Variáveis sensíveis marcadas `sensitive`
- [ ] `terraform plan` limpo, sem drift inesperado
- [ ] Recursos versionados/tagueados de forma consistente

Evidência/observações:

## 6. Observabilidade

- [ ] Logs/métricas mínimas definidas no TRD estão implementadas

Evidência/observações:

## 7. Pendências (acumulado entre fatias — nunca recriado do zero)

Uma linha por pendência, nunca removida — ao resolver, atualize o `Status` da própria linha em
vez de apagá-la ou duplicá-la. Pendência com `Status = resolvido` cuja `Fatia` já foi mergeada é
referência sob demanda, não leitura obrigatória nas rodadas seguintes (`docs/AGENT-PREAMBLE.md`,
seção "Artefato in-place que acumulou rodadas"). Pendência `aberto` continua leitura obrigatória
todo round, mesmo que de uma fatia anterior.

| # | Fatia | Item | Responsável | Bloqueia release? | Status |
|---|-------|------|--------------|---------------------|--------|

## 8. Pendências de validação (VALIDAR DEPOIS)

| ID    | Pergunta                          | Contexto                              | Status              |
|-------|--------------------------------------|-------------------------------------------|------------------------|
| SRE-1 | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>            | pendente / validado |

## 9. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 10. Próximo passo

Se aprovado: esta fatia está pronta para merge em `main` (`GIT-WORKFLOW.md` do pipeline) — se houver
fatias seguintes na feature, elas só começam depois deste merge. Caso contrário: itens acima devem
ser resolvidos antes.
