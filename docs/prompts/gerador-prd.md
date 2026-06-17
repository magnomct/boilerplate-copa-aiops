# Geração de PRD a partir do Brainstorm

Transforme o resultado do nosso brainstorm em um PRD (Product
Requirements Document) enxuto. Use o contexto consolidado da conversa
como única fonte de verdade. Não invente requisito novo nem reabra
decisão já fechada.

## Como agir

1. **Parta do brainstorm.** Releia o problema, as decisões em tabela
   e os pontos que consolidamos e construa o PRD em cima disso. Se já
   existe código ou contexto no projeto, investigue antes de assumir.

2. **Fique no "o quê", não no "como".** O PRD define o problema, as
   capacidades e os critérios de sucesso. Decisão de implementação fica
   para a spec (OpenSpec), não aqui.

3. **Não preencha lacuna com suposição.** Se algo essencial ficou em
   aberto no brainstorm, marque como "A definir" em vez de inventar.

4. **Escreva verificável.** Nada de requisito vago: "ser rápido" vira
   "responder em até X". Cada requisito deve poder virar um cenário de
   spec depois.

## Estrutura do PRD

Gere um documento em Markdown com estas seções:

1. **Visão geral** — 2-3 linhas: o que é e que problema resolve.
2. **Problema e contexto** — a dor que justifica o trabalho.
3. **Objetivos** — os resultados esperados (não tarefas).
4. **Fora de escopo** — o que explicitamente não entra.
5. **Capacidades e requisitos** — o que a solução precisa fazer, cada
   item observável, sem detalhe de implementação.
6. **Critérios de aceite** — como saber que cada capacidade está pronta.
7. **Restrições e premissas** — limites técnicos, de tempo ou de contexto.

## Saída

Apresente o PRD completo e, ao final, pergunte o que eu quero ajustar
antes de seguir para a spec no OpenSpec. Se faltar informação para
alguma seção, escreva "A definir"; não complete com achismo.