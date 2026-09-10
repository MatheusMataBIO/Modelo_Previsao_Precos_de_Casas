# Ficha do modelo

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

**Versão:** avaliacao_v1. **Algoritmo:** LightGBM (`LGBMRegressor`). **Estado:** avaliado no teste temporal; sem implantação.

## Uso e interface

Prever preços de imóveis com características compatíveis com o histórico fornecido. O pipeline `artifacts/modelo_avaliado.joblib` recebe 50 entradas enriquecidas e derivadas, na ordem registrada em `artifacts/model_card.json`. A integração da junção por CEP e da criação de features é descrita na proposta documental de empacotamento. Saída em USD, por premissa adotada no projeto, sem conversão cambial.

## Dados, treinamento e versões

17.191 vendas de treino, anteriores a 10/03/2015; teste com 4.331 vendas até 27/05/2015, sem compartilhamento de IDs. As 91 vendas antigas com IDs do teste foram excluídas. O modelo não foi reajustado com o teste e é o mesmo utilizado nas 100 previsões futuras.

Configuração selecionada: selected_candidate_v2. São 18 características físicas, sete derivadas e 25 indicadores demográficos, sem hous_val_amt. Imputação mediana, codificação one-hot de CEP e modelo treinado no log1p do alvo; expm1 retorna ao preço. Parâmetros e versões exatos constam no JSON, com hashes para rastrear os resultados.

## Métricas de teste

MAE de USD 74.089,41; MAPE de 12,77%; RMSE 129.025,96; R² 0,8768; erro mediano 44.655,82; 49,92% dentro de ±10% e 80,56% dentro de ±20%. Viés médio −45.704,70. Nenhuma previsão de teste não positiva. Redução de MAE de 27,48% frente ao Ridge, mas aumento de 19,02% frente ao MAE médio de validação.

## Limitações e uso responsável dos resultados

Não há tolerância de erro aprovada. Há subestimação, especialmente em imóveis acima de 1 milhão. Os indicadores demográficos precisam de confirmação da fonte e do período. A EDA anterior observou todo o histórico. Permutação e SHAP descrevem o modelo, não causalidade. Teste de um período/região não garante desempenho em outros contextos. Os futuros têm sobreposição de entradas com o histórico e não são uma avaliação independente.

Consulte [o relatório de avaliação](evaluation_summary.md) para métricas por segmento, explicabilidade, protocolo de reprodução e artefatos. A decisão de implantação depende dessas limitações e das necessidades dos stakeholders.


## Contexto do erro verificado na auditoria

**Complemento da auditoria — escala do erro e diferença fora do treino**

No teste, o preço médio é **554.588,88** e a mediana é **465.000,00**. O MAE de **74.089,41** equivale a **13,36% do preço médio**. Essa razão ajuda a entender a escala, mas não é MAPE nem significa que cada casa teve esse erro percentual. O erro percentual mediano por imóvel foi **10,03%**. Os valores são apresentados em USD, por premissa do projeto, sem conversão cambial.

Ao reaplicar o modelo salvo às próprias vendas de treino, o MAE foi **41.185,22**. A média de validação foi **62.249,43** e o teste foi **74.089,41**. O desempenho piora fora do treino; não foi demonstrada ausência de overfitting. Mudanças de período e composição dos imóveis também podem contribuir, por isso esses números não identificam sozinhos a causa.

Não há tolerância de negócio aprovada. Melhorar o Ridge não basta para aceitar precificação automática, especialmente com a subestimação dos imóveis caros. O registro das verificações já realizadas está na [auditoria do projeto](project_audit.md).


### MAPE: erro percentual médio como complemento

**Resultado no teste:** LightGBM teve **MAPE de 12,77%**, contra **20,17% do Ridge**, nas mesmas 4.331 vendas. Para cada imóvel, dividimos o erro absoluto pelo preço real, multiplicamos por 100 e depois calculamos a média. Isso expressa o tamanho médio do erro relativo; não significa 87,23% de acerto nem uma garantia para cada previsão.

**Por que incluir:** o MAE de USD 74.089,41 informa a escala monetária; o MAPE facilita comparar erros relativos entre preços diferentes. Ele também é diferente dos **10,03% de erro percentual mediano** e dos **13,36% da razão entre MAE e preço médio**.

**Cuidados e decisão:** o mesmo erro em USD pesa mais em imóveis baratos. Preços próximos de zero podem distorcer o MAPE; no teste, o menor preço é USD 81.000. Mantemos **MAE como critério principal de seleção**. MAPE foi acrescentado às previsões existentes, sem novos treinamentos ou escolha pelo teste; não há tolerância de negócio aprovada.

Na validação do candidato final, a média do MAPE das três janelas é **11,97%**. Agrupando todas as vendas de validação, é **11,92%**: as quantidades de vendas por janela diferem. Para comparar períodos com peso igual, usamos o primeiro valor.


### SHAP global e local

O notebook 06 agora inclui SHAP global nas mesmas 600 vendas usadas na permutação, além dos três exemplos locais. Latitude, `grade` e área habitável lideram a média de |SHAP|, com 0,1384, 0,1099 e 0,1037 em log(1 + preço). As barras mostram magnitude média; o beeswarm mostra direção e distribuição das contribuições. Não são efeitos causais, percentuais nem valores monetários.

O cálculo usa contribuições nativas do LightGBM e gráficos Matplotlib. O CEP é agrupado pela soma das contribuições one-hot por imóvel. As previsões e métricas não mudaram; nenhum modelo foi retreinado. Detalhes e gráficos em [SHAP global](shap_global.md).
