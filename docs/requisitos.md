# Especificação de Requisitos: Supermarket Receipt

## 1. Requisitos Funcionais (RF)

| Identificador | Categoria | Descrição / Regra de Negócio | Critério de Aceitação (Verificável) |
| :--- | :--- | :--- | :--- |
| **RF01** | Catálogo | Consultar produtos por nome, retornando preço unitário de tabela e unidade de medida (`EACH` ou `KILO`). | Consulta a `unit_price(produto)` retorna o valor exato cadastrado no catálogo sem erros. |
| **RF02** | Carrinho | Adicionar produto vendido por unidade com quantidade padrão de 1.0 item. | Chamada a `add_item(p)` inclui o item com quantidade 1.0 no histórico do carrinho. |
| **RF03** | Carrinho | Adicionar produto com quantidade inteira ou fracionária/peso em quilos (`add_item_quantity`). | Permite múltiplos registros e acumula corretamente o total em `product_quantities[p]`. |
| **RF04** | Oferta Legada | Aplicar percentual de desconto paramétrico sobre o total do produto (`TEN_PERCENT_DISCOUNT`). | Abatimento de $P\%$ sobre o subtotal do item; gera `Discount` com descrição `"{P}% off"`. |
| **RF05** | Oferta Legada | Aplicar promoção "Leve 3, Pague 2" (`THREE_FOR_TWO`) a cada grupo de 3 unidades do mesmo produto. | A cada 3 unidades inteiras ($\lfloor qtd / 3 \rfloor$), abate o valor de 1 unidade; descrição `"3 for 2"`. |
| **RF06** | Oferta Legada | Cobrar valor fixo promocional a cada par de itens (`TWO_FOR_AMOUNT`). | A cada 2 unidades inteiras, aplica o preço fechado; unidades ímpares excedentes mantêm preço normal. Descrição `"2 for {arg}"`. |
| **RF07** | Oferta Legada | Cobrar valor fixo promocional a cada conjunto de 5 unidades (`FIVE_FOR_AMOUNT`). | A cada 5 unidades inteiras, aplica o preço fechado; excedentes mantêm preço unitário normal. Descrição `"5 for {arg}"`. |
| **RF08** | Combo | Cadastrar ofertas de combo formadas por conjuntos definidos de produtos distintos (ex: escova + creme dental). | Permite registrar um combo contendo lista de produtos participantes e percentual de 10%. |
| **RF09** | Combo | Aplicar desconto de 10% exclusivamente sobre os itens que formam combos fechados completos. | O número de combos é $\min(\lfloor qtd \rfloor)$ dos itens. 10% incide apenas sobre a soma dos itens dos combos completos. |
| **RF10** | Convivência | Prevenir desconto duplicado sobre a mesma unidade de produto e preservar ofertas preexistentes. | Unidades alocadas no combo não recebem outra promoção. Itens excedentes e produtos fora do combo mantêm suas ofertas individuais normais. |
| **RF11** | Checkout | Processar fechamento da compra (`checks_out_articles_from`), gerando o recibo com itens, descontos e total líquido. | $\text{Total Líquido} = \sum \text{total\_price (itens)} - \sum \text{descontos}$. Válido para carrinho vazio (0.00) ou com itens. |
| **RF12** | Recibo Texto | Formatar e imprimir cupom fiscal em texto com 40 colunas alinhadas (`ReceiptPrinter`). | Linhas de itens com subtotal à direita, quantidades fracionárias com 3 casas (`%.3f`), descontos negativos e linha final `"Total: "`. |
| **RF13** | Extensibilidade | Permitir apresentação alternativa do recibo (ex: HTML) preservando os mesmos cálculos contábeis. | Separação clara entre modelo contábil (`Receipt`) e geradores de saída visual. |

---

## 2. Requisitos Não Funcionais (RNF)

| Identificador | Categoria | Descrição | Métrica / Critério Verificável de Sucesso |
| :--- | :--- | :--- | :--- |
| **RNF01** | Compatibilidade | Preservar as assinaturas e o contrato das interfaces públicas existentes (`ShoppingCart`, `Teller`, `Receipt`, `Product`). | 100% de sucesso nos testes de caracterização sem alterações na API pública. |
| **RNF02** | Design / SOLID | Eliminar excesso de condicionais em `handle_offers` utilizando padrões GoF: **Strategy** (descontos) e **Factory Method** (criação). | Complexidade ciclomática de cada estratégia $\le 5$; ausência de `if/elif` monolítico no carrinho. |
| **RNF03** | Segurança | Auditar o código e dependências antes e após a refatoração com **Bandit**, **pip-audit** e **Safety**. | 0 vulnerabilidades de severidade Alta ou Crítica; relatório de auditoria e justificativas documentadas. |
| **RNF04** | Testabilidade | Garantir cobertura automatizada de testes com **pytest** para comportamento legado e novos combos. | Suítes executando de forma rápida e determinística (sem I/O de rede/banco); cobertura de código $\ge 90\%$. |
| **RNF05** | Integridade Numérica | Garantir exatidão monetária em operações de multiplicação, divisão e percentuais. | Tolerância máxima de erro contábil de $\pm 0.001$; validação com `pytest.approx(..., 0.01)`. |
| **RNF06** | Processo MPS.BR | Manter rastreabilidade e critérios verificáveis entre requisitos, código, testes e evidências de segurança. | Matriz de rastreabilidade 100% preenchida vinculando cada RF/RNF a arquivos de teste e implementação. |

---

## 3. Critérios de Aceitação para Combos Promocionais

| Cenário | Situação de Compra | Comportamento Esperado | Critério de Aceitação Verificável |
| :--- | :--- | :--- | :--- |
| **Combo Completo** | 1 escova (R$ 10,00) + 1 creme dental (R$ 15,00) | Aplica 10% sobre o valor dos dois produtos (R$ 25,00). | Desconto de R$ 2,50; Total da compra = R$ 22,50. |
| **Combo Incompleto** | Somente 1 escova (R$ 10,00) | Não aplica desconto de combo. | Desconto de combo = R$ 0,00; Total da compra = R$ 10,00. |
| **Combos Repetidos** | 2 escovas + 2 cremes dentais | Aplica 10% sobre os 2 combos completos (R$ 50,00). | Desconto de R$ 5,00 ($2 \times \text{R\$} 2,50$); Total = R$ 45,00. |
| **Combo com Excedente** | 2 escovas + 1 creme dental | Aplica desconto em 1 combo; a escova excedente mantém preço normal. | Desconto de R$ 2,50; Total = R$ 32,50 (R$ 22,50 combo + R$ 10,00 escova). |
| **Convivência (Outro Produto)** | 1 combo (escova + creme) + 3 sabonetes ("3 por 2") | Aplica desconto do combo e aplica promoção individual dos sabonetes. | Desconto do combo (R$ 2,50) + desconto de 1 sabonete grátis calculados sem interferência mútua. |
| **Convivência (Mesmo Produto)** | 2 escovas (com oferta 10% individual) + 1 creme | 1 escova integra o combo; a escova excedente recebe o desconto individual. | Escova do combo recebe apenas desconto do combo; escova excedente recebe desconto individual. Zero duplo desconto. |

---

## 4. Matriz de Rastreabilidade

| Requisito | Descrição Sintética | Módulo / Componente | Arquivo de Teste (`pytest`) | Critério de Aceitação |
| :--- | :--- | :--- | :--- | :--- |
| **RF01** | Catálogo e Preços | `SupermarketCatalog` | `tests/test_comportamento_legado.py` | Consulta com preço e unidade válidos |
| **RF02** | Item Unitário Padrão | `ShoppingCart.add_item` | `tests/test_comportamento_legado.py` | Item inserido com quantidade 1.0 |
| **RF03** | Itens com Qtd / Peso | `ShoppingCart.add_item_quantity` | `tests/test_comportamento_legado.py` | Saldo correto em `product_quantities` |
| **RF04** | Desconto Percentual | `TenPercentDiscountStrategy` | `tests/test_comportamento_legado.py` | Abatimento de $P\%$ do valor do item |
| **RF05** | Oferta Leve 3, Pague 2 | `ThreeForTwoStrategy` | `tests/test_comportamento_legado.py` | 1 item grátis a cada 3 unidades inteiras |
| **RF06** | Oferta 2 por Valor Fixo | `TwoForAmountStrategy` | `tests/test_comportamento_legado.py` | Preço fechado no par; excedente integral |
| **RF07** | Oferta 5 por Valor Fixo | `FiveForAmountStrategy` | `tests/test_comportamento_legado.py` | Preço fechado no quinteto; excedente integral |
| **RF08** | Cadastro de Combos | `Teller`, `ComboOffer` | `tests/test_combos.py` | Registro do combo no catálogo/teller |
| **RF09** | Desconto em Combos | `BundleDiscountStrategy` | `tests/test_combos.py` | 10% sobre combos completos formados |
| **RF10** | Convivência de Ofertas | `OfferManager` / `Teller` | `tests/test_combos.py` | Sem desconto duplo em nenhuma unidade |
| **RF11** | Fechamento de Recibo | `Teller.checks_out_articles_from` | `tests/test_comportamento_legado.py` | Total líquido = soma itens - descontos |
| **RF12** | Formatação Texto Recibo | `ReceiptPrinter` | `tests/test_comportamento_legado.py` | Colunas alinhadas em 40 caracteres |
| **RF13** | Extensibilidade Recibo | `ReceiptPrinter` / `HtmlPrinter` | `tests/test_receipt_printer.py` | Modelo desacoplado da apresentação |
| **RNF01** | Retrocompatibilidade | Interfaces Públicas Legadas | `tests/test_comportamento_legado.py` | 100% dos testes legados aprovados |
| **RNF02** | Padrões Strategy / Factory | Módulos em `src/strategies/` | Inspeção estática e unitária | Ausência de condicionais em cascata |
| **RNF03** | Auditoria de Segurança | CI / Bandit, pip-audit, Safety | `docs/seguranca_antes_depois.md` | Zero alertas de severidade alta/crítica |
| **RNF04** | Cobertura de Testes | Suíte Pytest (`tests/`) | `pytest --cov=src` | Cobertura $\ge 90\%$ |
| **RNF05** | Integridade Numérica | Operações de Cálculo de Preço | `tests/test_comportamento_legado.py` | Precisão $\le \pm 0.001$ (`pytest.approx`) |
| **RNF06** | Processo e Rastreabilidade | Artefatos em `docs/` | Validação documental e auditoria | Rastreabilidade total RF/RNF $\leftrightarrow$ Testes |
