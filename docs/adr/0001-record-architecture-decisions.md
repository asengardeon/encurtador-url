# ADR 0001 — Registrar decisões de arquitetura como ADR

> Status: aceito

## Contexto

Decisões técnicas significativas (escolha de padrão, troca de tecnologia, trade-off de
arquitetura) tendem a ficar apenas na cabeça de quem decidiu, ou espalhadas em comentários de
código, e se perdem com o tempo — quem chega depois refaz a mesma discussão sem saber que ela já
aconteceu.

## Decisão

Toda decisão técnica significativa identificada pelo agente `architect` (ou por qualquer pessoa
trabalhando no repositório) é registrada como um ADR (Architecture Decision Record) em
`docs/adr/NNNN-titulo-curto.md`, numerado sequencialmente, seguindo esta estrutura:

```markdown
# ADR NNNN — <título curto no imperativo>

> Status: proposto | aceito | substituído por ADR NNNN

## Contexto

Qual problema/força levou a precisar decidir algo aqui.

## Decisão

O que foi decidido.

## Alternativas consideradas

Outras opções e por que não foram escolhidas.

## Consequências

O que fica mais fácil, o que fica mais difícil, o que passa a ser uma restrição a partir de
agora.
```

Um ADR não é editado depois de aceito para "consertar" uma decisão — se a decisão muda, cria-se
um novo ADR que substitui o anterior (`status: substituído por ADR NNNN` no antigo).

## Alternativas consideradas

- Deixar decisões só na descrição do PR: descartado, PRs são efêmeros e difíceis de buscar depois.
- Wiki externa: descartado, decisão de arquitetura deve viver junto do código que ela afeta.

## Consequências

- Todo TRD que envolve uma decisão não óbvia referencia um ADR.
- Ganha-se histórico consultável de "por que fizemos assim" sem depender de memória de quem
  participou da decisão.
