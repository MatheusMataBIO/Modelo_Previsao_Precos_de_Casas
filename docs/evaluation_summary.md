# Avaliação final e explicabilidade

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

O notebook `06_avaliacao_explicabilidade.ipynb` avalia a configuração LightGBM congelada no notebook 05, compara com Ridge e gera as previsões dos 100 exemplos futuros. Não houve busca adicional de parâmetros, seleção de colunas pelo teste ou ajuste com as vendas do teste.

## Dados e protocolo

Treino final: 17.191 vendas de 02/05/2014 a 09/03/2015. Teste: 4.331 vendas de 10/03/2015 a 27/05/2015. As 91 vendas antigas cujos imóveis aparecem no teste permanecem excluídas. Treino e teste não compartilham IDs. Dados, ordem das linhas, versões e hashes da configuração foram conferidos.

O LightGBM usa as 50 entradas e os parâmetros selecionados no 05, com log1p do preço e expm1 nas previsões. Ridge usa suas 18 entradas e alpha 1.0, solver lsqr e tolerância 1e-6. Cada preparação aprende apenas no treino final. O ganho entre modelos combina algoritmo, entradas e parâmetros.

## Resultados no teste

| Métrica | Ridge | LightGBM |
|---|---:|---:|
| MAE (USD) | 102.170,13 | 74.089,41 |
| RMSE (USD) | 172.212,44 | 129.025,96 |
| MAPE | 20,17% | 12,77% |
| R² | 0,7805 | 0,8768 |
| Erro absoluto mediano | 72.542,24 | 44.655,82 |
| Percentil 90 do erro absoluto | 193.378,84 | 158.094,36 |
| Erro percentual mediano | 15,28% | 10,03% |
| Dentro de ±10% | 33,92% | 49,92% |
| Dentro de ±20% | 61,37% | 80,56% |
| Viés médio (previsto − real) | −36.637,55 | −45.704,70 |
| Previsões não positivas | 13 | 0 |

O LightGBM reduziu o MAE em 27,48% frente ao Ridge. Seu MAE ficou 19,02% acima da média da validação anterior, de 62.249,43. Não são períodos nem tamanhos de treino idênticos; essa diferença não identifica sozinha a causa. Não há uma tolerância de erro aprovada pelo negócio. ±10% e ±20% são referências descritivas, não critérios de aprovação.

## Segmentos e falhas

Nos 315 imóveis acima de 1 milhão, o MAE foi 278.380,19 e apenas 63,81% das previsões ficaram dentro de ±20%. Na faixa de 300 a 600 mil, com 2.178 vendas, o MAE foi 50.307,25 e a proporção dentro de ±20% foi 84,57%.

As 32 vendas com waterfront = 1 tiveram erro percentual mediano de 24,65%; as 4.299 com valor zero, 9,97%. Os grupos têm preços e características diferentes; não é uma comparação causal. Grupos de grade com pouquíssimas vendas não permitem conclusões gerais.

Os dez maiores erros absolutos foram subestimações de imóveis acima de 1 milhão. O maior erro foi de aproximadamente 1,78 milhão. Nenhuma dessas vendas foi removida para melhorar o resultado. Hipóteses de baixa representatividade, mudança de mercado e características ausentes exigem investigação adicional, sem retunar no mesmo teste.

## Explicabilidade

A permutação usou uma amostra aleatória de 600 vendas do teste, três repetições e MAE na escala original. Latitude, área habitável e grade lideraram o ranking. Correlações e combinações artificiais entre originais, derivadas e indicadores do CEP limitam a interpretação; baixa importância não autoriza remover uma entrada. [Referência oficial](https://scikit-learn.org/stable/modules/permutation_importance.html).

As contribuições SHAP nativas do LightGBM explicam três exemplos escolhidos pelos percentis 10, 50 e 90 do preço previsto. São contribuições na escala log1p do preço, não valores monetários. A soma com o valor de referência, seguida de expm1, reproduziu as previsões. [Referência oficial](https://lightgbm.readthedocs.io/en/stable/pythonapi/lightgbm.Booster.html#lightgbm.Booster.predict).

## Artefatos e reprodução

- `artifacts/modelo_avaliado.joblib`: pipeline LightGBM efetivamente avaliado, sem ajuste com teste.
- `artifacts/ridge_avaliado.joblib`: baseline avaliado no mesmo período.
- `reports/metrics/avaliacao_registro.json`: assinatura dos dados/configuração, hashes dos artefatos e registro de avaliação.
- `reports/metrics/avaliacao_*.csv`: métricas, segmentos, importâncias e explicações.
- `reports/predictions/avaliacao_teste.csv`: preços reais e previsões de teste, com linha de origem.
- `reports/predictions/future_unseen_predictions.csv`: 100 previsões, ordem e entradas originais preservadas.
- `artifacts/model_card.json`: ficha técnica estruturada, versões e origem dos arquivos.

O ajuste dos dois modelos, previsão e exportação inicial levou cerca de 0,85 segundo; não é o tempo total do notebook. Usamos até duas threads. Reexecutar o notebook com a mesma assinatura reaproveita modelos e previsões; não constitui nova avaliação independente. Mudanças de configuração ou fontes interrompem o reaproveitamento em vez de sobrescrever silenciosamente o registro.

## Limites e continuidade

A EDA examinou o histórico completo. A origem e a disponibilidade da demografia na época das vendas continuam pendentes. O teste representa um período e uma região, não todos os cenários futuros. Os exemplos futuros compartilham características com o histórico e não fornecem preços reais para avaliação.

O joblib recebe as 50 features preparadas. O notebook também verifica a junção por CEP e as derivadas dos futuros, mas esse preparo ainda é externo ao artefato. A etapa 07 documenta como empacotar a preparação completa e o modelo para uma API; a implementação do serviço fica fora do escopo acordado. A avaliação não equivale a autorização de implantação ou aprovação pelo negócio.


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

**Leitura por faixa:** o MAPE foi **15,08%** até USD 300 mil; **11,46%** de USD 300 a 600 mil; **12,29%** de USD 600 mil a 1 milhão; e **17,39%** acima de USD 1 milhão. O erro relativo também é maior no grupo mais caro. As faixas são definidas pelo preço real e servem à avaliação retrospectiva, não para classificar automaticamente o risco de uma previsão futura.
