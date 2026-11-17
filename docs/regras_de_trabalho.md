## Regras Gerais do Trabalho

O trabalho deverá ser realizado em grupo, conforme orientação do professor. Cada grupo entregará o pacote completo de modernização do projeto, com documentação, código-fonte, testes e evidências reproduzíveis.

### Restrições obrigatórias

* Preservar as regras comerciais existentes: cálculos, ofertas, descontos e valores exibidos no recibo.
* Preservar o contrato observável das interfaces públicas existentes; mudanças internas devem ser justificadas.
* Não substituir integralmente o programa por uma solução nova e independente do legado.
* Implementar a promoção de combos de produtos e cobrir seus casos normais e de borda.
* Incorporar ao menos dois padrões de projeto com aplicação real e justificativa técnica; padrões artificiais não serão valorizados.
* Executar Bandit, pip-audit e Safety, registrar resultados e apresentar a análise das ocorrências.
* Priorizar testes de caracterização antes de alterações que possam modificar regras de negócio.

### Nova regra de negócio — combos promocionais

Um combo promocional contém um conjunto definido de produtos. Quando o cliente adquire todos os produtos necessários a um combo, aplica-se desconto de 10% apenas sobre o valor dos itens que integram os combos completos.

| Situação de compra | Comportamento esperado |
| :--- | :--- |
| 1 escova + 1 creme dental | Aplicar 10% sobre o valor dos dois produtos. |
| 2 escovas + 1 creme dental | Aplicar desconto apenas em 1 combo; a escova excedente mantém preço normal. |
| 2 escovas + 2 cremes dentais | Aplicar desconto em 2 combos completos. |
| Somente 1 escova | Não aplicar desconto de combo. |
| Carrinho com promoções preexistentes | Manter o comportamento anterior e evitar desconto indevido ou duplicado no mesmo conjunto de itens. |

> **Exemplo ilustrativo:** escova R$ 10,00 + creme dental R$ 15,00 = R$ 25,00. Um combo completo recebe desconto de R$ 2,50 e totaliza R$ 22,50. Os valores do repositório podem ser diferentes; o critério obrigatório é a regra dos 10%.

A equipe deverá explicitar na documentação o tratamento de múltiplos combos e o critério de convivência com outras ofertas; esses cenários deverão ser cobertos por testes.
