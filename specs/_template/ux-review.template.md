# UX Review — <Nome da Feature>

> Autor: agente `ux-designer`
> PRD: `specs/<slug>/prd.md` | TRD: `specs/<slug>/trd.md`
> Fatia validada nesta rodada: `<F-1, ou "única — feature não fatiada">`
> PR desta fatia: <link do Pull Request revisado>
> Método desta rodada: `<verificação ao vivo (navegação real) | leitura de código, sem automação de navegador disponível>`
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

Uma linha por rodada de UX review (uma por fatia) — nunca sobrescreva o veredito de uma fatia já
aprovada e mergeada, acrescente uma linha nova.

| Fatia | PR      | Veredito | Método | Data |
|-------|---------|----------|--------|------|
| F-1   | `<link>` | `<veredito>` | `<ao vivo/leitura>` | `<data>` |

## 2. Consistência e padrões

Tokens/componentes do design system (`docs/DESIGN-SYSTEM.md`, se existir) seguidos, ou padrão ad
hoc introduzido sem justificativa? Achado (arquivo/componente, esperado, observado, severidade) ou
"sem achados".

## 3. Visibilidade do estado do sistema e prevenção de erro

Ação destrutiva/irreversível com confirmação; estados de carregamento/vazio/erro com feedback
visual próprio. Achado ou "sem achados".

## 4. Controle e liberdade do usuário / navegabilidade

Saída clara de todo fluxo (cancelar/voltar); convenções de navegação (ex.: logo → home);
alinhamento de navegação global entre viewports. Achado ou "sem achados".

## 5. Consciência de estado/ciclo de vida

Ações condicionadas a um estado do domínio (ex.: "ativo" vs. "encerrado") adaptadas/ocultadas
corretamente. Achado ou "sem achados".

## 6. Robustez de conteúdo gerado pelo usuário

Fallback visual para upload/mídia quebrada. Achado ou "sem achados".

## 7. Hierarquia de informação

Peso visual de ações condizente com risco/privilégio (ex.: conta pessoal vs. administração).
Achado ou "sem achados".

## 8. Alvo de toque/acessibilidade básica

Tamanho mínimo de alvo interativo, contraste, navegação por teclado/foco visível. Achado ou "sem
achados", ou "não verificável nesta rodada" com o motivo (ex.: sem automação de navegador
disponível).

## 9. Achados (se reprovado ou aprovado com ressalvas)

| # | Arquivo/componente | Esperado | Observado | Severidade |
|---|----------------------|----------|-----------|------------|

## 10. Pendências de validação (VALIDAR DEPOIS)

Ambiguidade de design não coberta pelo PRD/TRD/`docs/DESIGN-SYSTEM.md` vira pergunta ao usuário;
se ele não souber responder agora, registre aqui em vez de decidir por conta própria. **ID no
formato `UX-<AAAA-MM-DD>-<slug-curto>`** (data + slug curto do próprio item — nunca um contador
sequencial simples) — mesmo critério de `docs/gates/licoes.md`/`docs/LESSONS-LEARNED.md` sobre
colisão de ID entre branches paralelas.

| ID                          | Pergunta                          | Contexto                    | Status              |
|------------------------------|--------------------------------------|-----------------------------------|------------------------|
| UX-2026-08-25-cor-de-alerta   | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>    | pendente / validado |

## 11. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 12. Próximo passo

`/sdd-qa` (se aprovado ou aprovado com ressalvas aceitas, para esta fatia) ou `/sdd-implement` (se
reprovado, com os achados acima).
