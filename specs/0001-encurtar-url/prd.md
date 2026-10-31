# PRD — Encurtar URL

> Status: aprovado
> Autor: agente `product-design`
> Relacionado: nenhuma (primeira spec do projeto)

## 1. Visão geral

Uma aplicação de encurtador de URL. O usuário cola uma URL longa em um campo, clica em um botão
de ação, e recebe uma versão encurtada que pode copiar e compartilhar.
**Escopo: uma spec é uma feature de usuário.** O objetivo de usuário único desta spec é encurtar
uma URL e poder usá-la/compartilhá-la — isso inclui o redirecionamento da URL encurtada para a
original, parte essencial do que "encurtar uma URL" significa. É a primeira e, por ora, única
feature do projeto. O pedido não reúne mais de um objetivo de usuário distinto.

Esta é a 1ª de uma sequência de 2 specs: **`0001-encurtar-url`** (esta spec) e
`0002-expirar-link` (reservada, ainda não iniciada). A capacidade de um link encurtado expirar
(ex.: depois de um tempo) foi desmembrada para a spec seguinte. Nesta spec, o link encurtado é
sempre permanente (ver seção 5, "Fora de escopo"). A spec `0002-expirar-link` só começa depois
desta fechar.

## 2. Problema

URLs longas são difíceis de compartilhar, lembrar e colar em contextos com limite de caracteres
(mensagens, redes sociais, materiais impressos). O usuário precisa de uma forma rápida de
transformar uma URL longa em uma versão curta que leve exatamente ao mesmo destino.

## 3. Usuários / Personas

Qualquer pessoa que precise compartilhar um link mais curto. O uso é totalmente anônimo: não há
login, cadastro, nem distinção de papel ou permissão. Qualquer visitante acessa a tela e encurta
URLs livremente, sem se identificar.

## 4. Objetivos

- Permitir que qualquer usuário encurte uma URL válida em poucos segundos, sem passos extras.
- Garantir que todo link encurtado gerado redirecione corretamente para a URL original enquanto
  estiver ativo.
- Oferecer uma forma de copiar o link encurtado sem exigir seleção manual de texto.

## 5. Fora de escopo

- Autenticação ou criação de conta de usuário. O uso é totalmente anônimo, sem papéis nem
  permissões (decisão do usuário).
- Escolha de um alias/código customizado pelo usuário. Nesta v1, o código é sempre gerado
  automaticamente pelo sistema. Pode entrar como melhoria futura, sem spec reservada ainda.
- Expiração automática do link encurtado. Nesta spec (`0001-encurtar-url`), todo link gerado é
  permanente. Essa capacidade fica inteiramente na spec seguinte da sequência,
  `0002-expirar-link` (reservada, ainda não iniciada) — não é "fora de escopo para sempre".
- Histórico ou painel de links criados por um mesmo usuário. Cada encurtamento é uma interação
  isolada; não há lista de "links recentes", nem em sessão de navegador (decisão do usuário).
- Deduplicação de URL já encurtada antes. Encurtar a mesma URL longa duas vezes gera um código
  novo a cada vez, nunca reaproveita um código já existente (decisão do usuário — ver US-1).
- Métricas de clique visíveis ao usuário (ex.: contador de acessos por link).
- Geração de QR code para o link encurtado.
- Acesso via API pública ou encurtamento de várias URLs em lote.
- Suporte a múltiplos domínios ou marcas de link encurtado.

## 6. Histórias de usuário e critérios de aceite

### US-1: Encurtar uma URL

Como usuário, eu quero colar uma URL longa e clicar em um botão de ação, para que eu receba uma
versão encurtada dessa URL.

**Critérios de aceite:**

```gherkin
Cenário: Encurtar uma URL válida
  Dado que o usuário está na página inicial com o campo de URL vazio
  Quando o usuário cola uma URL válida no campo e clica no botão de encurtar
  Então o sistema exibe uma URL encurtada correspondente na tela

Cenário: Tentar encurtar com o campo vazio
  Dado que o campo de URL está vazio
  Quando o usuário clica no botão de encurtar
  Então o sistema exibe uma mensagem informando que o campo é obrigatório
  E o sistema não gera nenhuma URL encurtada

Cenário: Tentar encurtar uma URL malformada
  Dado que o usuário digitou um texto que não é uma URL válida
  Quando o usuário clica no botão de encurtar
  Então o sistema exibe uma mensagem de erro informando que a URL é inválida
  E o sistema não gera nenhuma URL encurtada

Cenário: Feedback de carregamento durante o encurtamento
  Dado que o usuário colou uma URL válida e clicou no botão de encurtar
  Quando o sistema ainda está processando o pedido
  Então a tela exibe um indicador visual de carregamento até o resultado ou o erro aparecer

Cenário: Encurtar a mesma URL mais de uma vez
  Dado que uma URL original já foi encurtada antes, gerando um código A
  Quando o usuário encurta a mesma URL original de novo
  Então o sistema gera um código novo, diferente de A, para a mesma URL original
```

### US-2: Copiar a URL encurtada

Como usuário, eu quero clicar em um botão de copiar ao lado da URL encurtada, para que eu possa
colá-la em outro lugar sem digitar manualmente.

**Critérios de aceite:**

```gherkin
Cenário: Copiar a URL encurtada com sucesso
  Dado que uma URL encurtada está exibida na tela
  Quando o usuário clica no botão de copiar
  Então a URL encurtada é copiada para a área de transferência do dispositivo
  E o sistema exibe uma confirmação visual de que a cópia foi bem-sucedida
```

### US-3: Acessar a URL encurtada

Como pessoa que recebe um link encurtado, eu quero acessar essa URL em um navegador, para que eu
seja levado à página original sem precisar conhecer a URL completa.

**Critérios de aceite:**

```gherkin
Cenário: Redirecionamento bem-sucedido
  Dado que uma URL encurtada válida foi gerada anteriormente para uma URL original
  Quando alguém acessa a URL encurtada em um navegador
  Então o navegador é redirecionado para a URL original correspondente

Cenário: Acesso a uma URL encurtada que não existe
  Dado que o código de uma URL encurtada não corresponde a nenhuma URL original cadastrada
  Quando alguém acessa essa URL encurtada
  Então o sistema exibe uma página informando que o link não foi encontrado
  E o sistema não redireciona para lugar nenhum
```

## 7. Wireframes/Protótipos de tela (quando a feature tem UI)

- **Oferecido ao usuário?** Sim.
- **Sistema de design usado**: `docs/DESIGN-SYSTEM.md` (estabelecido nesta rodada — paleta azul
  `#2563eb` + neutros de cinza, tom minimalista, tipografia sans-serif padrão sem preferência
  específica; o arquivo não existia antes desta rodada).
- **Opções apresentadas**: 3 opções de baixa fidelidade via skill `design` — Opção A (card
  centralizado, layout vertical único), Opção B (barra horizontal estilo hero/bit.ly), Opção C
  (painel dividido com proposta de valor à esquerda). Artifact publicado:
  https://claude.ai/code/artifact/aef5b974-23d3-4ab9-a499-72b10e44fd7e
- **Opção escolhida**: Opção A — card centralizado. O usuário escolheu esta direção visual entre
  as três apresentadas.
- **Arquivo salvo**: `specs/0001-encurtar-url/wireframes/opcao-a-centrado.dc.html` (opção
  escolhida). As demais opções também foram salvas para referência futura:
  `specs/0001-encurtar-url/wireframes/opcao-b-barra-horizontal.dc.html` e
  `specs/0001-encurtar-url/wireframes/opcao-c-painel-dividido.dc.html`, além do
  `specs/0001-encurtar-url/wireframes/canvas.json` com o estado do canvas de design.
- **Parecer de UX (`ux-designer`)**: coletado em modo consultoria sobre as 3 opções. Resumo
  transcrito: a Opção A tem o melhor equilíbrio entre simplicidade, hierarquia de informação e
  feedback visual consistente de estado — o mesmo tratamento de cor entre a caixa de ação/input e
  a caixa de resultado — sem observações bloqueantes. A Opção B tem feedback de estado de sucesso
  mais fraco, porque o resultado aparece sem destaque de cor. A Opção C é mais pesada visualmente e
  levanta perguntas abertas sobre o painel estático em usos recorrentes e sobre a responsividade em
  viewport estreito; nenhuma delas é bloqueante, são só pontos de atenção para o TRD/implementação
  caso essas opções fossem escolhidas — o que não é o caso aqui. Nota de método do `ux-designer`:
  nenhum wireframe de baixa fidelidade desenha os estados dinâmicos (carregando, "copiado com
  sucesso", URL inválida). Esses estados ficam cobertos pelos critérios de aceite das histórias
  US-1 e US-2 deste PRD, e não representam falha de nenhuma das três opções.

## 8. Métricas de sucesso

Nenhuma meta numérica concreta foi fornecida pelo usuário até o momento; as métricas abaixo
descrevem o que a aplicação deve permitir observar depois do lançamento, para apoiar a definição
de metas concretas mais adiante:

- Taxa de sucesso do encurtamento: proporção de tentativas com URL válida que resultam em uma URL
  encurtada exibida sem erro inesperado.
- Taxa de uso do botão de copiar: proporção de encurtamentos bem-sucedidos seguidos do clique no
  botão de copiar, como indício de que o resultado foi útil.
- Taxa de redirecionamento bem-sucedido: proporção de acessos a URLs encurtadas que resultam em
  redirecionamento correto, em vez de erro de link não encontrado.

## 9. Indicadores técnicos a observar

- **Volumetria**: relevante. O usuário espera uso público de alto tráfego, não um projeto pessoal
  de baixo uso. O `architect` deve considerar, no TRD, a escala de leitura/escrita do mapeamento
  código→URL (ex.: estratégia de cache, capacidade do armazenamento, taxa de geração de códigos
  sem colisão) — o PRD só sinaliza o requisito, não desenha a solução.
- **Segurança**: indicador relevante. Um encurtador de URL expõe uma superfície de
  redirecionamento aberto: qualquer código encurtado pode levar a qualquer destino, incluindo
  sites de phishing ou malware, se o sistema não validar nem moderar o destino de alguma forma.
  O uso anônimo combinado com alto tráfego esperado também traz risco de abuso na criação de
  links em massa (spam), o que pode exigir alguma forma de limite de taxa — decisão técnica do
  `architect`, não deste PRD. Não há coleta de dados pessoais nesta versão, porque o uso é
  totalmente anônimo.
- **Legal/compliance**: nenhum indicador relevante identificado. O uso é anônimo e não há
  conta de usuário nem coleta de dado pessoal nesta spec.

## 10. Ordem de valor / dependências entre histórias (fatias verticais de entrega)

| Fatia | Histórias/critérios cobertos | Depende de | Motivo | O que fica demonstrável ao final desta fatia |
|-------|-------------------------------|------------|--------|-------------------------------------------------|
| F-1   | US-1 (todos os cenários: feliz, campo vazio, URL malformada, carregamento, reencurtar a mesma URL) | nenhuma | ponto de entrada da feature; validação de entrada e geração de código novo a cada encurtamento fazem parte do mesmo formulário e não entregam valor isoladas do caminho feliz | Usuário cola uma URL válida, clica em "encurtar" e vê a URL encurtada na tela (um código novo mesmo se repetir a mesma URL); ou vê uma mensagem de erro clara ao tentar encurtar algo inválido/vazio. |
| F-2   | US-2 (copiar com feedback de sucesso) | F-1 (precisa existir um resultado exibido para copiar) | incremento fino sobre o resultado já exibido em F-1 | Usuário, após encurtar uma URL, clica em "copiar" e vê a confirmação visual de que o texto foi copiado para a área de transferência. |
| F-3   | US-3 (redirecionamento bem-sucedido e link não encontrado) | F-1 (precisa existir o mapeamento persistido entre código curto e URL original) | fecha o ciclo de utilidade da URL encurtada — sem isso ela não leva a lugar nenhum | Alguém acessa, em um navegador, a URL encurtada gerada em F-1 e é redirecionado para a URL original; acessar um código inexistente mostra página de "não encontrado". |

## 11. Pendências de validação (VALIDAR DEPOIS)

Nenhuma pendência aberta. As 7 pendências originais (PRD-1 a PRD-7) foram todas resolvidas pelo
usuário e incorporadas às seções correspondentes deste PRD (3, 5, 6, 9 e 1).

| ID     | Pergunta                          | Contexto                              | Status              |
|--------|--------------------------------------|-------------------------------------------|------------------------|

## 12. Log de revisões

| Data | Autor | O que mudou | Motivo | Etapas revalidadas |
|------|-------|--------------|--------|-----------------------|
| 2026-10-30 | orquestrador (sem agente) + `product-design` | Resolvidas as 7 pendências PRD-1 a PRD-7: uso anônimo sem papéis (PRD-1); entidade única sem categorização, expiração desmembrada para `0002-expirar-link` reservada (PRD-2); código novo a cada encurtamento, sem deduplicação (PRD-3); sem alias customizado na v1 (PRD-4); link permanente nesta spec (PRD-5); uso público de alto tráfego, indicador de volumetria/segurança adicionado à seção 9 (PRD-6); sem histórico de links (PRD-7). | Respostas explícitas do usuário coletadas via `AskUserQuestion` pelo orquestrador, já que `product-design` rodou em background sem acesso a essa ferramenta. | Seções 1, 3, 5, 6, 9 revalidadas contra as respostas. |

## 13. Aprovação

- [x] Aprovado por: usuário em 2026-10-30
