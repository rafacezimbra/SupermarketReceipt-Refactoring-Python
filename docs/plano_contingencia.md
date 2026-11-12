# Plano de Contingência e Refatoração: Supermarket Receipt

## 1. Identificação de Riscos, Medidas Preventivas e Mitigação

| ID | Descrição do Risco | Requisitos Impactados | Prob. / Impacto | Medida Preventiva / Mitigação | Teste Associado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R01** | **Regressão nos cálculos das ofertas legadas** ao desacoplar `handle_offers`. | RF04, RF05, RF06, RF07, RNF01 | Média / Alto | Criar suíte completa de testes de caracterização cobrindo 100% das ofertas antes de alterar o código de produção. | `tests/test_comportamento_legado.py` |
| **R02** | **Quebra de interfaces públicas existentes** consumidas por clientes legados. | RNF01, RF02, RF03, RF11 | Baixa / Alto | Preservar as assinaturas de `ShoppingCart`, `Teller`, `Receipt` e `Product`. Manter métodos legados delegando internamente aos novos padrões. | `tests/test_comportamento_legado.py` |
| **R03** | **Duplo desconto no mesmo item** (conflito entre combo e promoção individual). | RF09, RF10 | Alta / Alto | Implementar política de alocação de saldo: unidades consumidas para fechar o combo são debitadas do saldo antes de avaliar ofertas individuais. | `tests/test_combos.py` |
| **R04** | **Falha de cálculo em combos incompletos ou com excedentes**. | RF08, RF09 | Média / Médio | Aplicar regra do gargalo: $N_{\text{combos}} = \min(\lfloor qtd \rfloor)$ dos itens participantes; unidades excedentes faturadas pelo preço unitário integral. | `tests/test_combos.py` |
| **R05** | **Imprecisão de ponto flutuante em totais e descontos**. | RNF05, RF11, RF12 | Média / Médio | Padronizar arredondamentos e validações com tolerância monetária rigorosa de $\pm 0.001$. | Testes com `pytest.approx(..., 0.01)` |
| **R06** | **Introdução de dependências vulneráveis ou código inseguro**. | RNF03 | Baixa / Alto | Estabelecer linha de base de segurança inicial com Bandit, pip-audit e Safety; repetir auditoria a cada entrega. | Scripts de CI e relatórios de segurança |

---

## 2. Invariantes de Negócio e Interfaces Públicas Imutáveis

| Elemento / Interface | Requisitos Vinculados | Contrato Imutável / Invariante de Negócio | Critério de Verificação |
| :--- | :--- | :--- | :--- |
| **`ShoppingCart.add_item(product)`** | RF02, RNF01 | Deve registrar exatamente 1.0 unidade do produto no carrinho. | `cart.items[-1].quantity == 1.0` |
| **`ShoppingCart.add_item_quantity(p, q)`** | RF03, RNF01 | Permite quantidades inteiras e fracionárias; acumula saldo em `product_quantities[p]`. | Saldo em `product_quantities` igual à soma exata das entradas. |
| **`ShoppingCart.handle_offers(...)`** | RF04-RF07, RNF01 | Mantém a assinatura pública legada intacta; delega internamente a execução de descontos. | Execução sem erro de tipo ou de assinatura. |
| **`Teller.checks_out_articles_from(cart)`**| RF11, RNF01 | Recebe o carrinho e retorna instância válida de `Receipt` contendo itens e descontos. | Tipo de retorno é `Receipt`; total consolidado correto. |
| **`Receipt.total_price()`** | RF11, RNF05 | $\text{Total Líquido} = \sum \text{total\_price (itens)} - \sum \text{descontos}$. | Tolerância contábil nula em relação aos itens e descontos. |
| **`ReceiptPrinter.print_receipt(receipt)`**| RF12, RNF01 | Gera recibo em texto com linhas formatadas em exatamente 40 colunas de largura. | Todas as linhas têm largura exata de 40 caracteres. |
| **Isolamento de Unidade Física** | RF10 | Uma mesma unidade física de produto só pode ser beneficiada por uma única promoção. | Inexistência de descontos cumulativos sobre a mesma unidade. |

---

## 3. Sequência Ordenada de Alterações (Refatoração Incremental)

| Ordem | Etapa da Modernização | Requisitos Cobertos | Pré-condição de Entrada | Ações de Implementação | Critério de Aceitação para Avançar |
| :---: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Linha de Base de Segurança** | RNF03 | Repositório original baixado | Executar Bandit, pip-audit e Safety no código inicial. Registrar achados na documentação. | Linha de base registrada em `docs/seguranca_antes_depois.md`. |
| **2** | **Testes de Caracterização** | RF01-RF07, RF11-RF12, RNF01, RNF04 | Ambiente virtual configurado | Criar testes automatizados para todas as regras legadas existentes sem alterar código de produção. | **100% dos testes de caracterização verdes** no pytest. |
| **3** | **Extração do Padrão Strategy** | RF04-RF07, RNF01, RNF02 | Passo 2 concluído e aprovado | Criar interface `OfferStrategy` e classes concretas para cada oferta legada (`ThreeForTwo`, `TenPercent`, etc.). | Testes de caracterização continuam 100% verdes sem quebras. |
| **4** | **Aplicação do Factory Method** | RNF01, RNF02 | Passo 3 concluído e aprovado | Implementar `OfferStrategyFactory` para instanciar a estratégia correspondente sem `if/elif` monolítico. | `ShoppingCart` delega para o Factory; complexidade ciclomática $\le 5$. |
| **5** | **Implementação dos Combos** | RF08, RF09, RF10 | Passo 4 concluído e aprovado | Criar `BundleDiscountStrategy` com desconto de 10% para combos completos e alocação de saldo. | Cenários de combo completo, incompleto, repetido e excedente testados. |
| **6** | **Testes de Regressão e Borda** | RF09, RF10, RNF04, RNF05 | Passo 5 concluído | Executar suíte completa unindo testes legados e novos testes de combos e convivência de ofertas. | **100% dos testes aprovados**; cobertura de código $\ge 90\%$. |
| **7** | **Reavaliação de Segurança** | RNF03, RNF06 | Passo 6 concluído | Reexecutar Bandit, pip-audit e Safety. Gerar comparativo antes/depois e justificar ocorrências. | Zero novas vulnerabilidades; relatório de auditoria finalizado. |

---

## 4. Estratégia de Reversão e Rollback

| Gatilho de Falha (Trigger) | Requisito Violado | Procedimento Operacional de Reversão | Ação no Git / Código | Teste de Confirmação Pós-Rollback |
| :--- | :--- | :--- | :--- | :--- |
| **Falha em teste de caracterização legado** durante a refatoração (Passos 3 ou 4). | RNF01, RF04-RF07 | Interromper a refatoração imediatamente. Identificar a alteração causadora da divergência de cálculo ou de contrato. | `git restore <arquivo>` ou `git revert HEAD` para retornar ao último commit verde. | Executar `pytest tests/test_comportamento_legado.py` até obter 100% verde. |
| **Erro de duplo desconto ou conflito de combo** (Passo 5). | RF09, RF10 | Isolar a classe `BundleDiscountStrategy` e reverter a integração no fluxo de ofertas do `Teller`/`ShoppingCart`. | Reverter a ramificação do commit de integração do combo (`git checkout <commit-passo-4>`). | Executar suíte legada garantindo que ofertas existentes continuam íntegras. |
| **Quebra de formatação no recibo** (`ReceiptPrinter`). | RF12, RNF01 | Restaurar a lógica de formatação de strings e espaçamento colunado original. | `git checkout HEAD~1 -- python_pytest/tests/receipt_printer.py` | Executar testes de comparação de impressão de recibo textual. |
| **Alerta crítico em auditoria de segurança** (Bandit / pip-audit / Safety). | RNF03 | Reverter dependência adicionada ou refatorar trecho apontado como vulnerável. | Desinstalar dependência insegura e atualizar `requirements.txt`. | Reexecutar a ferramenta de auditoria até zerar alertas críticos. |

---

## 5. Critérios de Aceitação e Checklist Go / No-Go

| Item de Verificação | Requisitos Relacionados | Comando / Ferramenta de Validação | Condição de Aprovação (Go / No-Go) |
| :--- | :--- | :--- | :--- |
| **Testes de Caracterização** | RF01-RF07, RF11-RF12, RNF01 | `pytest tests/test_comportamento_legado.py` | **Obrigatório: 0 falhas**. Nenhuma regressão de cálculo ou contrato público permitida. |
| **Testes de Combos e Borda** | RF08, RF09, RF10 | `pytest tests/test_combos.py` | **Obrigatório: 0 falhas**. Cobre combo simples, incompleto, repetido, excedente e convivência. |
| **Cobertura de Código** | RNF04 | `pytest --cov=src --cov-report=term-missing` | **Cobertura $\ge 90\%$** nas classes de negócio (`src/`). |
| **Auditoria Bandit** | RNF03 | `bandit -r src/` | **0 vulnerabilidades de severidade Alta ou Média**. |
| **Auditoria de Dependências** | RNF03 | `pip-audit` e `safety check` | **0 CVEs críticas** nos pacotes instalados. |
| **Controle de Versão** | RNF06 | `git status` e `git log --oneline` | Commits atômicos, mensagens semânticas e árvore de trabalho limpa. |
