# PRD — <Nome da Feature>

> Status: rascunho | em revisão | aprovado
> Autor: agente `product-design`
> Relacionado: (link para specs anteriores, se houver)

## 1. Visão geral

Uma a três frases: o que é a feature e por que ela existe agora.
**Escopo: uma spec é uma feature de usuário.** Nomeie aqui, em uma linha, o **objetivo de usuário
único** que esta spec atende. Se o pedido original reunia mais de um, registre a sequência de specs
em que ele foi quebrado e qual é a posição desta — ex.: "2ª de 3: `0007-convite-de-membro`,
**`0008-permissao-por-papel`**, `0009-auditoria-de-acesso`". A spec seguinte da sequência só começa
depois desta fechar, igual ao que já vale entre fatias. Critério de quebra e o custo que ele evita:
`agents/product-design.md`, passo 1b.

## 2. Problema

Qual problema real do usuário/negócio isto resolve. Evite descrever a solução aqui — só o
problema.

## 3. Usuários / Personas

Quem usa isso e em que contexto.

## 4. Objetivos

- Objetivo mensurável 1
- Objetivo mensurável 2

## 5. Fora de escopo

O que explicitamente NÃO está incluído nesta feature (evita escopo implícito mal-entendido).

## 6. Histórias de usuário e critérios de aceite

### US-1: <título curto>

Como <persona>, eu quero <ação>, para que <benefício>.

**Critérios de aceite:**

```gherkin
Cenário: <nome>
  Dado <contexto>
  Quando <ação>
  Então <resultado esperado>
```

(Repita para cada história. Cada critério de aceite deve ser verificável por um terceiro sem
contexto adicional — isso vira o checklist literal do QA.)

## 7. Wireframes/Protótipos de tela (quando a feature tem UI)

Preenchido só quando a feature tem tela(s) nova(s) ou mudança visual relevante em tela existente.
Antes de escrever as histórias de usuário em detalhe, `/sdd-prd` oferece ao usuário ver 2-3 opções
de wireframe/protótipo de baixa fidelidade das telas principais para escolher a direção que achar
mais interessante — uma exploração visual rápida para alinhar direção cedo, não o desenho técnico
de UI final (isso continua com `architect`/`frontend-developer` no TRD/implementação).

- **Oferecido ao usuário?** `<sim / não aplicável (feature sem UI)>`
- **Sistema de design usado**: `<"docs/DESIGN-SYSTEM.md" (já existia) / "docs/DESIGN-SYSTEM.md"
  (estabelecido nesta rodada) / "paleta neutra padrão — usuário optou por não estabelecer um
  sistema de design agora" / "não aplicável (wireframes não oferecidos)">`
- **Opções apresentadas**: `<link do Artifact com as opções, ou "nenhuma — usuário preferiu seguir
  direto para o PRD">`
- **Opção escolhida**: `<qual opção, e por quê, se o usuário comentou>`
- **Arquivo salvo**: `specs/<slug>/wireframes/<nome-descritivo>.dc.html` — a fonte local das
  opções geradas, salva junto da spec para conferência futura mesmo se o Artifact publicado não
  estiver mais acessível (republique esse arquivo via Artifact para ver de novo). `<preencher ou
  "não aplicável">`
- **Parecer de UX (`ux-designer`)**: consultoria de navegabilidade/usabilidade coletada por
  `/sdd-prd` (passo 2b) antes das histórias serem escritas em detalhe — por opção de wireframe
  quando houve mais de uma, ou sobre a descrição textual das telas/fluxos quando o usuário optou
  por não ver wireframes. Informativo, não um veredito — não bloqueia a aprovação do PRD.
  `<transcrever o parecer recebido, "sem observações relevantes", ou "não aplicável (feature sem
  alteração de UI)">`

## 8. Métricas de sucesso

Como saberemos que a feature funcionou depois de lançada.

## 9. Indicadores técnicos a observar

Não é papel do PRD decidir arquitetura, mas é papel do PRD **sinalizar** o que pode ter peso
técnico significativo, para o `architect` endereçar no TRD. Preencha cada item mesmo que a
resposta seja "nenhum" — não deixe implícito.

- **Volumetria**: ordem de grandeza de dados/tráfego esperado (registros, requisições/segundo,
  crescimento esperado). `<preencher ou "nenhum indicador relevante">`
- **Segurança**: envolve dado sensível/pessoal, autenticação, autorização, ou superfície de
  ataque nova? `<preencher ou "nenhum indicador relevante">`
- **Legal/compliance**: implica LGPD/GDPR, retenção de dados, contrato com terceiro, ou
  regulação específica do domínio? `<preencher ou "nenhum indicador relevante">`

## 10. Ordem de valor / dependências entre histórias (fatias verticais de entrega)

Visão de produto (não técnica) de como a feature será entregue em **fatias verticais**: cada
fatia é um incremento fino que atravessa toda a pilha necessária para ela (nunca "todo o backend
primeiro, frontend depois" quando a feature é full-stack) e entrega algo demonstrável/testável de
ponta a ponta, ainda que mínimo. É isso que permite mostrar a entrega da spec completa **aos
poucos**, em vez de só no final. Uma fatia pode cobrir uma história inteira, parte de uma
história, ou combinar partes de histórias diferentes — o critério é "dá para demonstrar/validar
algo real ao final desta fatia", não "isto é uma camada arquitetural" (ex.: "construir toda a
API" não é uma fatia vertical válida). O `architect` usa esta tabela como ponto de partida para a
decomposição técnica no TRD, que pode adicionar dependências que só a arquitetura revela — mas
deve preservar o caráter demonstrável de cada fatia na implementação.

| Fatia | Histórias/critérios cobertos | Depende de | Motivo | O que fica demonstrável ao final desta fatia |
|-------|-------------------------------|------------|--------|-------------------------------------------------|
| F-1   | US-1 (cenário feliz)            | nenhuma     | ponto de entrada da feature | `<ex.: usuário consegue criar uma tarefa e vê-la na lista, de ponta a ponta>` |

## 11. Pendências de validação (VALIDAR DEPOIS)

Toda ambiguidade que o usuário não soube/quis responder agora, registrada aqui em vez de virar
suposição silenciosa. Resolvida via `/sdd-amend` quando o usuário pedir a revisão.

| ID     | Pergunta                          | Contexto                              | Status              |
|--------|--------------------------------------|-------------------------------------------|------------------------|
| PRD-1  | <pergunta que ficou sem resposta>     | <por que essa pergunta importa>            | pendente / validado |

## 12. Log de revisões

Preenchido pelo `/sdd-amend` a cada mudança neste PRD depois de aprovado — nunca recrie o
documento do zero para registrar uma mudança.

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|

## 13. Aprovação

- [ ] Aprovado por: <usuário> em <data>
